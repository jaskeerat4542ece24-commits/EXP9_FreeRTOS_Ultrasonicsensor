# STM32 FreeRTOS Inter-Task Communication using Queues

This repository contains an embedded C project developed for the STM32 Nucleo-F446RE development board. The project demonstrates the implementation of thread-safe inter-task communication using FreeRTOS Queues via the CMSIS-RTOS v2 API, using an ultrasonic sensor data pipeline as a practical application.

---

## Project Overview

The objective of this project is to shift away from vulnerable global variables and implement a robust Producer-Consumer architectural pattern for data sharing between concurrent threads.

* **Kernel Framework:** FreeRTOS managed via the CMSIS-RTOS V2 wrapper layer.
* **Communication Primitive:** FreeRTOS Message Queue (utilizing copy-by-value structural integrity).
* **Core Architecture (Producer-Consumer):**
  * **The Producer (Sensor Task):** Periodically acquires distance readings from an HC-SR04 Ultrasonic Sensor every 1000 ms and posts the data to the queue using `osMessageQueuePut`.
  * **The Consumer (Processing Task):** Blocks on the queue via `osMessageQueueGet`. It consumes zero CPU cycles while waiting, immediately waking up only when a new data packet arrives to process or display the data.

---

## Queue Theory and Architecture

Unlike basic arrays, FreeRTOS queues are specifically engineered for concurrency and multi-tasking environments:

* **Copy-by-Value:** Data sent to the queue is physically copied into kernel-managed memory slots. This ensures that the original variable can safely go out of scope or be overwritten by the sensor task without corrupting the value waiting to be processed by the consumer.
* **Thread Safety:** The kernel automatically manages critical section locking during queue reads and writes to eliminate data race conditions.
* **Natural Back-Pressure:** If the consumer falls behind and the queue fills up, the producer can be configured to block, preventing data loss and matching the production rate to the consumption capability.

---

## Hardware Requirements

* Development Board: STM32 Nucleo-F446RE (ARM Cortex-M4)
* External Components: HC-SR04 Ultrasonic Distance Sensor, Breadboard, and Jumper wires
* Connectivity: USB Type-A to Mini-B cable (supporting SWD and Trace capabilities)

---

## Hardware & Peripheral Configuration

The system initialization profile is established inside STM32CubeMX with these parameter mappings:

| Component / Interface | MCU Pin | Configuration Mode | Function |
| :--- | :--- | :--- | :--- |
| **HC-SR04 TRIG** | `PA9` | GPIO Output | Triggers ultrasonic burst |
| **HC-SR04 ECHO** | `PA10` | Input Capture / GPIO | Captures return pulse width |
| **SYS Debug Mode** | N/A | Trace Asynchronous Sw | Enables SWV ITM Console Output |
| **SYS Timebase Source** | N/A | Hardware Timer (`TIM6`) | Isolated HAL Tick Source |

### Critical System Core Adjustments
* **System Clock:** Configured via internal oscillator (RCC Bypass mode) running at a stable **84 MHz**.
* **Timebase Source Selection:** The HAL timebase source is shifted from `SysTick` to hardware timer **TIM6**. FreeRTOS claims exclusive rights to `SysTick` for its 1 ms slicing clock, making this alteration necessary to prevent kernel resource crashes.

---

## FreeRTOS Queue & Task Attributes

The communication queue and tasks are instantiated inside the CMSIS-RTOS v2 middleware configuration block:

### Queue Specification
* **Queue Name:** `sensorQueueHandle`
* **Queue Slots:** 10 elements
* **Element Size:** `uint32_t` (or custom data structures)

### Task Specifications
* **Sensor Task (Producer):** Priority level set to `osPriorityNormal`, executes with a periodic loop cycle of 1000 ms.
* **Processing Task (Consumer):** Priority level set to `osPriorityNormal`, remains blocked until an element is pushed into the queue buffer.

---

## Core Code Implementation

### The Producer: Sensor Task
```c
void StartSensorTask(void *argument)
{
  uint32_t distance_cm = 0;
  for(;;)
  {
    // [Insert hardware-specific HC-SR04 trigger and pulse measurement logic here]
    distance_cm = read_ultrasonic_distance(); 

    // Push data copy onto the queue with a 0-millisecond block time timeout limit
    osMessageQueuePut(sensorQueueHandle, &distance_cm, 0U, 0U);
    
    // Periodic tracking rate delay
    osDelay(1000);
  }
}
