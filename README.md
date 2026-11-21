# HC-SR04-Ultrasonic-Sensor-with-STM32-and-LCD-Timer-Input-Capture-


## 📌 Project Overview

In this project, we connect an **HC-SR04 ultrasonic distance sensor** to an **STM32F103C8T6 (Blue Pill)** microcontroller and display the measured distance on a **16x2 LCD**.

The main goal of this project is not just to measure distance, but also to understand how STM32 timers work, especially the **Input Capture mode**.

Instead of using `HAL_Delay()` or continuously checking the Echo pin to measure pulse duration, we use **TIM1 Input Capture with Interrupts** to detect the rising and falling edges of the Echo signal.

This project is a practical introduction to:

- Ultrasonic distance measurement
- STM32 Timer configuration
- Input Capture mode
- Interrupt-based programming
- Measuring pulse width
- Timer overflow handling
- LCD interfacing using STM32 HAL

---

## 🛠️ Hardware Requirements

| Component | Description |
|---|---|
| STM32F103C8T6 | Blue Pill development board |
| HC-SR04 | Ultrasonic distance sensor |
| 16x2 LCD | Character LCD display |
| Jumper wires | For connections |
| Breadboard | Optional |

## 🔌 Pin Connections

### HC-SR04 to STM32

| HC-SR04 Pin | STM32 Pin | Description |
|---|---|---|
| VCC | 5V | Power supply |
| GND | GND | Common ground |
| TRIG | PA6 | Trigger signal (Output) |
| ECHO | PA8 | Echo signal (TIM1_CH1) |

**Important:** The HC-SR04 Echo pin produces a 5V signal, while STM32F103 GPIO pins operate at 3.3V logic. Use an appropriate voltage divider or level shifter to protect the STM32 input.

### LCD to STM32

| LCD Pin | STM32 Pin |
|---|---|
| RS | PA0 |
| EN | PA1 |
| D4 | PA2 |
| D5 | PA3 |
| D6 | PA4 |
| D7 | PA5 |

The LCD is controlled using 4-bit communication mode.

*Note: The LCD connections are based on the pin configuration used in this project.*

---

## ⚙️ How Does the HC-SR04 Work?

The HC-SR04 measures distance by transmitting ultrasonic sound waves and measuring the time it takes for the reflected sound to return.

The measurement process consists of four main steps:

1. The STM32 sends a HIGH pulse of at least 10 microseconds to the TRIG pin.
2. The HC-SR04 transmits an ultrasonic burst at approximately 40 kHz.
3. When the reflected sound is detected, the ECHO pin goes HIGH for a duration proportional to the distance.
4. The STM32 measures the HIGH pulse duration and calculates the distance.

### Understanding the Echo Signal

The Echo pulse width represents the total time required for the ultrasonic wave to travel to the object and return.

Therefore, we must divide the measured travel distance by two when calculating the actual distance to the object.

The distance formula is:

\[
Distance = \frac{Time \times Speed\ of\ Sound}{2}
\]

For the HC-SR04, a commonly used practical formula is:

\[
Distance(cm) = \frac{Echo\ Time(\mu s)}{58}
\]

Why 58?

The speed of sound in air is approximately 343 m/s, or 0.0343 cm/µs.

Considering the round trip:

\[
Distance = \frac{Time \times 0.0343}{2}
\]

This gives approximately:

\[
Distance(cm) = \frac{Time(\mu s)}{58.3}
\]

In our code, we use the simplified divisor of 58.

---

## ⏱️ Understanding STM32 Timer Input Capture

This is the most important part of the project.

### 1. Why Input Capture?

We need to measure how long the Echo signal stays HIGH.

One possible approach is to continuously read the GPIO pin and use software delays. However, this method is not ideal for accurate timing.

Instead, STM32 timers provide a hardware feature called **Input Capture**.

Input Capture allows the timer to automatically save its current counter value when a specific signal edge is detected.

This means we don't need to manually read the timer at exactly the right moment.

### 2. TIM1 Configuration

We configure TIM1 as follows:

| Parameter | Configuration |
|---|---|
| Timer | TIM1 |
| Channel | Channel 1 |
| Mode | Input Capture |
| Input Selection | Direct Input |
| Initial Polarity | Rising Edge |
| Prescaler | 7 |
| Counter Period (ARR) | 65535 |
| Interrupt | Enabled |

The timer input clock is configured based on the 8 MHz HSE clock.

The prescaler calculation is:

\[
Timer\ Frequency = \frac{8MHz}{7+1}
\]

\[
Timer\ Frequency = 1MHz
\]

Therefore:

\[
Timer\ Tick = 1\mu s
\]

Each increment of the timer counter represents one microsecond.

This makes measuring the Echo pulse duration much easier.

### 3. Detecting the Rising Edge

Initially, TIM1 Channel 1 is configured to detect the rising edge of the Echo signal.

When the Echo signal changes from LOW to HIGH, the timer captures its current counter value.

We save this value in:

`rising_value`

We then change the capture polarity to falling edge:

`__HAL_TIM_SET_CAPTUREPOLARITY()`

Now the timer waits for the Echo signal to return to LOW.

### 4. Detecting the Falling Edge

When the Echo signal changes from HIGH to LOW, another Input Capture event occurs.

The timer automatically captures its counter value again.

We save this value in:

`falling_value`

Now we have both timestamps:

- `rising_value`: Timer count when Echo becomes HIGH.
- `falling_value`: Timer count when Echo becomes LOW.

The Echo pulse duration can now be calculated.

\[
Echo\ Time = Falling\ Value - Rising\ Value
\]

However, there is one important detail!

### 5. Handling Timer Overflow

TIM1 is configured with:

`ARR = 65535`

This means the counter counts from 0 to 65535 and then starts again from 0.

What happens if the timer overflows between the rising and falling edges?

For example:

- Rising value = 65000
- Falling value = 500

A simple subtraction would produce an incorrect result.

We must consider the counter overflow:

\[
Echo\ Ticks = (65536 - Rising) + Falling
\]

In the code, we handle both situations:

- If the falling value is greater than or equal to the rising value, we subtract normally.
- Otherwise, we calculate the elapsed time considering the timer overflow.

This ensures that the measured pulse duration remains correct even when the counter wraps around.

---

## ⚡ Understanding Interrupts

Instead of continuously checking the Echo signal in the main loop, we use an interrupt.

The following HAL callback is called when an Input Capture event occurs:

`HAL_TIM_IC_CaptureCallback()`

Inside this callback, we:

1. Check whether the interrupt belongs to TIM1 Channel 1.
2. Detect whether this is the first capture or the second capture.
3. Save the rising or falling capture value.
4. Change the capture polarity.
5. Calculate the Echo pulse duration.
6. Set `measurement_ready = 1` when the measurement is complete.

### Why Use a Flag?

Notice that we don't update the LCD directly inside the interrupt callback.

Instead, we set:

`measurement_ready = 1`

The main loop checks this flag and updates the LCD when a complete measurement is available.

This approach separates time-critical capture operations from display operations.

It also helps keep the interrupt callback short and efficient.

---

## 💻 Understanding the Main Loop

### 1. Starting Input Capture

We start TIM1 Channel 1 with interrupts enabled:

`HAL_TIM_IC_Start_IT(&htim1, TIM_CHANNEL_1);`

This allows the timer to detect the configured input edges and generate interrupts.

### 2. Generating the Trigger Pulse

To start a measurement, we generate a pulse on PA6:

- LOW for 2 µs
- HIGH for 10 µs
- LOW again

The 10 µs HIGH pulse triggers the HC-SR04 measurement process.

### 3. Calculating the Distance

When the measurement is ready, we calculate the distance:

`dis = echo_ticks / 58.0f;`

We use a floating-point variable to keep decimal values.

### 4. Displaying the Result

Finally, we format the measured distance using `sprintf()`:

`sprintf(data, "Dist:%7.2f cm", dis);`

Then we send the resulting string to the LCD:

`lcd_puts(data);`

The LCD displays the measured distance in centimeters.

---

## 📚 Key Concepts Learned

By completing this project, you can practice several important embedded systems concepts:

| Concept | What You Learn |
|---|---|
| GPIO | Configuring input and output pins |
| Timer Prescaler | Adjusting the timer counting frequency |
| Auto Reload Register | Defining the timer counting range |
| Input Capture | Capturing timer values on signal edges |
| Interrupts | Responding to hardware events |
| Rising Edge | Detecting LOW-to-HIGH transitions |
| Falling Edge | Detecting HIGH-to-LOW transitions |
| Timer Overflow | Handling counter rollover |
| Pulse Width Measurement | Measuring how long a signal stays HIGH |
| HAL Library | Working with STM32 HAL functions |
| LCD Interfacing | Displaying measurement results |

---

## 🚀 Final Result

The STM32 generates a trigger pulse, the HC-SR04 measures the distance using ultrasonic waves, and TIM1 Input Capture measures the Echo pulse duration.

The calculated distance is then displayed on the 16x2 LCD in centimeters.

This project is a useful starting point for understanding hardware timers and interrupt-driven programming in STM32 microcontrollers.

---

## 🎥 YouTube Tutorial

**Let's Connect HC-SR04 to STM32 and Display Distance on an LCD!**

Watch the complete tutorial to understand the wiring, timer configuration, Input Capture mechanism, and distance calculation step by step.

---

## 👩‍💻 Author

**Ellis Learning Journey**

Learning embedded systems through practical projects, experiments, and datasheet-based development.

GitHub: https://github.com/elli-codes

YouTube: https://www.youtube.com/@ellislearningjourney 
Full tutorial :https://youtube.com/shorts/wXk62y6QKkw?feature=share 
