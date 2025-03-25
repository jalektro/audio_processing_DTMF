#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "arm_math.h"
#include "audio.h"
#include "xscutimer.h"
#include "xscugic.h"
#include "math.h"

#define UINT32_MAX_AS_FLOAT 4294967295.0f
#define UINT_SCALED_MAX_VALUE 0xFFFFF

#define TIMER_DEVICE_ID      XPAR_XSCUTIMER_0_DEVICE_ID
#define INTC_DEVICE_ID       XPAR_SCUGIC_SINGLE_DEVICE_ID
#define TIMER_IRPT_INTR      XPAR_SCUTIMER_INTR

#define SAMPLE_RATE          48000
#define TONE_DURATION        (0.5 * SAMPLE_RATE)  // Duration of the tone in samples
#define PAUSE_DURATION       (0.3 * SAMPLE_RATE)  // Duration of the pause in samples
#define REPEAT_PAUSE         (2 * SAMPLE_RATE)    // Delay before repeating the sequence in samples

// DTMF Frequencies for digits 0-9
const float32_t dtmf_low_freq[] = {941, 697, 697, 697, 770, 770, 770, 852, 852, 852};
const float32_t dtmf_high_freq[] = {1336, 1209, 1336, 1477, 1209, 1336, 1477, 1209, 1336, 1477};

char sequence[] = "35846"; // Sequence of digits to generate
int sequence_length = sizeof(sequence) - 1;
volatile int current_digit = 0;
volatile int tone_timer = 0;
volatile int pause_timer = 0;
volatile int repeat_timer = 0;

static void Timer_ISR(void * CallBackRef) {
    static float32_t theta1 = 0.0f, theta2 = 0.0f;
    float32_t theta_increment1, theta_increment2;
    XScuTimer *TimerInstancePtr = (XScuTimer *) CallBackRef;

    if (tone_timer > 0) {
        int digit = sequence[current_digit] - '0';
        float32_t frequency1 = dtmf_low_freq[digit];
        float32_t frequency2 = dtmf_high_freq[digit];

        theta_increment1 = 2 * PI * frequency1 / SAMPLE_RATE;
        theta_increment2 = 2 * PI * frequency2 / SAMPLE_RATE;

        theta1 += theta_increment1;
        if (theta1 > 2 * PI) theta1 -= 2 * PI;
        theta2 += theta_increment2;
        if (theta2 > 2 * PI) theta2 -= 2 * PI;

        float32_t sine_value1 = arm_sin_f32(theta1);
        float32_t sine_value2 = arm_sin_f32(theta2);
        float32_t combined_sine_value = (sine_value1 + sine_value2) / 2.0f;
        uint32_t scaled_value = (uint32_t)(((combined_sine_value + 1.0f) * 0.5f) * UINT_SCALED_MAX_VALUE);
        Xil_Out32(I2S_DATA_TX_L_REG, scaled_value);  // Output to left channel
        Xil_Out32(I2S_DATA_TX_R_REG, scaled_value);  // Output to right channel
        tone_timer--;
    } else if (pause_timer > 0) {
        Xil_Out32(I2S_DATA_TX_L_REG, 0x800000);
        Xil_Out32(I2S_DATA_TX_R_REG, 0x800000);
        pause_timer--;
    } else if (repeat_timer > 0) {
        // Output silence during the repeat pause
        Xil_Out32(I2S_DATA_TX_L_REG, 0x800000);
        Xil_Out32(I2S_DATA_TX_R_REG, 0x800000);
        repeat_timer--;
    } else {
        if (current_digit >= sequence_length - 1) {
            current_digit = 0; // Restart the sequence
            repeat_timer = REPEAT_PAUSE; // Wait before repeating the sequence
        } else {
            current_digit++;
            tone_timer = TONE_DURATION;
            pause_timer = PAUSE_DURATION;
        }
    }

    XScuTimer_ClearInterruptStatus(TimerInstancePtr);
}

static int Timer_Intr_Setup(XScuGic * IntcInstancePtr, XScuTimer *TimerInstancePtr, u16 TimerIntrId) {
    int Status;
    XScuGic_Config *IntcConfig = XScuGic_LookupConfig(INTC_DEVICE_ID);
    if (!IntcConfig) {
        xil_printf("Failed to find GIC config\n");
        return XST_FAILURE;
    }
    Status = XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig, IntcConfig->CpuBaseAddress);
    if (Status != XST_SUCCESS) {
        xil_printf("GIC Initialization Failed\n");
        return Status;
    }
    Xil_ExceptionInit();
    Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_IRQ_INT, (Xil_ExceptionHandler)XScuGic_InterruptHandler, IntcInstancePtr);
    Status = XScuGic_Connect(IntcInstancePtr, TimerIntrId, (Xil_ExceptionHandler)Timer_ISR, (void *)TimerInstancePtr);
    if (Status != XST_SUCCESS) {
        xil_printf("Failed to connect ISR\n");
        return Status;
    }
    XScuGic_Enable(IntcInstancePtr, TimerIntrId);
    XScuTimer_EnableInterrupt(TimerInstancePtr);
    Xil_ExceptionEnable();
    return XST_SUCCESS;
}

int main() {
    int Status;
    init_platform();
    XScuTimer TimerInstance;
    XScuGic IntcInstance;

    XScuTimer_Config *TimerConfig = XScuTimer_LookupConfig(TIMER_DEVICE_ID);
    if (!TimerConfig) {
        xil_printf("Failed to find timer config\n");
        return XST_FAILURE;
    }
    Status = XScuTimer_CfgInitialize(&TimerInstance, TimerConfig, TimerConfig->BaseAddr);
    if (Status != XST_SUCCESS) {
        xil_printf("Timer Initialization Failed\n");
        return Status;
    }
    Status = Timer_Intr_Setup(&IntcInstance, &TimerInstance, TIMER_IRPT_INTR);
    if (Status != XST_SUCCESS) {
        xil_printf("Timer Interrupt Setup Failed\n");
        return Status;
    }
    XScuTimer_LoadTimer(&TimerInstance, XPAR_PS7_CORTEXA9_0_CPU_CLK_FREQ_HZ / 2 / SAMPLE_RATE);
    XScuTimer_EnableAutoReload(&TimerInstance);
    XScuTimer_Start(&TimerInstance);
    tone_timer = TONE_DURATION;
    pause_timer = PAUSE_DURATION;
        for(;;) {
            // The infinite loop keeps the program running
        }
        cleanup_platform();
        return 0;
    }
