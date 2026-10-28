# STM32 Accelerometer Motion Classification with Edge Impulse

On-device motion classification on an **STM32 Nucleo-F401RE** (ARM Cortex-M4). An **ADXL345** accelerometer is read over **SPI**, and a quantized TinyML model trained in **Edge Impulse** classifies the motion as **motion** or **stationary**. Inference runs entirely on the microcontroller with no cloud connection.

> Demo video: [ADD LINK]

## What it does

1. The STM32 configures the ADXL345 over SPI and reads X, Y and Z acceleration every 20 ms (50 Hz).
2. Raw `x,y,z` values are printed as CSV over UART, which can be used to collect training data.
3. Samples are collected into a window of 53 samples x 3 axes (about 1.06 s). [CONFIRM: `main.c` fills `features[]` with this window before each `run_classifier()` call]
4. The Edge Impulse spectral-analysis block turns each window into 39 features, and a neural network scores the two classes.
5. The label and score for each class are printed over UART.

## Hardware

| Part | Details |
|---|---|
| Board | STM32 Nucleo-F401RE (STM32F401RET6, ARM Cortex-M4, LQFP64) |
| Sensor | ADXL345 3-axis accelerometer, full resolution, +/-2 g |
| Sensor interface | SPI1 (master, mode 3: CPOL high, CPHA 2nd edge, 8-bit, MSB first) |
| SPI clock | 250 kHz (16 MHz / 64) |
| Serial output | USART2, 115200 baud, 8N1, through the ST-LINK virtual COM port |
| System clock | Internal 16 MHz HSI, no PLL |

### Wiring

| ADXL345 pin | Nucleo-F401RE pin |
|---|---|
| VCC | 3.3 V |
| GND | GND |
| CS | PA4 (software chip select, GPIO output) |
| SCL / SCLK | PA5 (SPI1_SCK) |
| SDA / SDI | PA7 (SPI1_MOSI) |
| SDO | PA6 (SPI1_MISO) |

USART2 TX/RX use PA2 and PA3, which are connected to the ST-LINK virtual COM port on the Nucleo board.

## ADXL345 configuration

| Register | Value | Meaning |
|---|---|---|
| `POWER_CTL` (0x2D) | 0x08 | Measurement mode |
| `DATA_FORMAT` (0x31) | 0x08 | Full resolution, +/-2 g |
| `DATAX0` (0x32) | 6-byte burst read | X, Y, Z as 16-bit values |

## Model

| Item | Value |
|---|---|
| Classes | `motion`, `stationary` |
| Input | 3 axes (ax, ay, az) at 50 Hz |
| Window | 53 samples per axis (about 1.06 s) |
| Signal processing | Spectral analysis (FFT length 16), 39 output features, standard-scaler normalization |
| Classifier | Neural network, int8 quantized, compiled with the EON compiler |
| Confidence threshold | 0.6 |
| TFLite tensor arena | 2,982 bytes (`EI_CLASSIFIER_TFLITE_LARGEST_ARENA_SIZE`) |
| Accuracy (test set) | 92 % |
| Inference latency on the STM32 | 30 ms |
| Flash / RAM used by the firmware | 150 KB	/ 20 KB |

## Repository structure

```
Core/Inc, Core/Src   Application code: main, ADXL345 SPI driver, inference loop, interrupt handlers
Drivers/             STM32 HAL and CMSIS drivers
edge-impulse-sdk/    Edge Impulse inferencing SDK
model-parameters/    Model metadata and DSP/normalization parameters exported from Edge Impulse
tflite-model/        Compiled neural network exported from Edge Impulse
*.ioc                STM32CubeMX configuration (SPI1, USART2, PA4 chip select)
*.ld                 Linker scripts (FLASH and RAM)
```

## How to build and run

1. Install **STM32CubeIDE**.
2. Clone this repository: `git clone https://github.com/veena110/stm32-imu-gesture-recognition.git`
3. In STM32CubeIDE, choose **File > Open Projects from File System** and select the cloned folder.
4. Connect the ADXL345 to the Nucleo board as in the wiring table.
5. Build the project and flash it with the on-board ST-LINK.
6. Open a serial terminal on the ST-LINK virtual COM port at 115200 baud. You will see the raw `x,y,z` values and the score for each class.

## How the model was made

1. Collected ADXL345 data for the `motion` and `stationary` classes.
2. Designed the impulse in Edge Impulse Studio: a spectral-analysis block followed by a classifier.
3. Trained and tested the model.
4. Exported it as a C++ library, which is included here in `edge-impulse-sdk`, `model-parameters` and `tflite-model`.
5. Integrated it into the STM32CubeIDE project and connected it to the SPI sensor driver.

## Limitations

- The model recognises two classes (`motion` and `stationary`), not named gestures such as swipes or shakes.
- The ADXL345 values are raw counts, so data used for inference must be scaled the same way as the training data.
- Results depend on how the sensor is mounted and how the data was collected.

## Licence and third-party code

See `LICENSE`. The Edge Impulse generated files (`edge-impulse-sdk`, `model-parameters`, `tflite-model`) are subject to the Edge Impulse Terms of Service, as stated in their file headers. The STM32 HAL and CMSIS files keep the licences of STMicroelectronics and Arm.

## Author

Veena S · [LinkedIn](https://www.linkedin.com/in/veena-shivanna-491b5b12b)
