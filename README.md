# Audio Codec (AK4954A)

### Overview

This library provides functions to support use of the AK4954A audio codec found on the CY8CKIT-028-TFT shield.

### Quick Start
1. Create an empty application.
2. Add this library to the application.
3. Add the following code to your application. <br/>
The _wave.h_ and _wave.c_ files can be pulled from this example: <br/>
https://github.com/Infineon/mtb-example-psoc6-i2s

NOTE: This example is targeted for Arduino-based boards.

```cpp
#include "cyhal.h"
#include "cybsp.h"
#include "mtb_ak4954a.h"
#include "wave.h"

#define AUDIO_SYS_CLOCK_HZ (98000000u)   /* in Hz (ideally 98.304 MHz) */
#define MCLK_FREQ_HZ       (4083000u)    /* in Hz (ideally 4.096 MHz) */
#define MCLK_DUTY_CYCLE    (50.0f)       /* in %  */

cyhal_clock_t pll_clock;
cyhal_clock_t audio_clock;
cyhal_clock_t system_clock;
cyhal_pwm_t mclk_pwm;
cyhal_i2s_t i2s;
cyhal_i2c_t i2c;

const cyhal_i2c_cfg_t i2c_config = {
    .is_slave        = false,
    .address         = 0,
    .frequencyhal_hz = 400000
};

const cyhal_i2s_pins_t i2s_pins = {
    .sck  = CYBSP_D1,
    .ws   = CYBSP_D2,
    .data = CYBSP_D3,
    .mclk = NC
};

const cyhal_i2s_config_t i2s_config = {
    .is_tx_slave    = false,    /* TX is Master */
    .is_rx_slave    = false,    /* RX not used */
    .mclk_hz        = 0,        /* External MCLK not used */
    .channel_length = 32,       /* In bits */
    .word_length    = 16,       /* In bits */
    .sample_rate_hz = 16000     /* In Hz */
};

void i2s_isr_handler(void *arg, cyhal_i2s_event_t event)
{
    (void) arg;
    (void) event;

    /* Stop the I2S TX */
    cyhal_i2s_stop_tx(&i2s);
}

int main(void)
{
    cy_rslt_t rslt;

    rslt = cybsp_init();
    if(CY_RSLT_SUCCESS != rslt)
    {
        CY_ASSERT(0);
    }

    __enable_irq();

    /* Initialize the PLL */
    rslt = cyhal_clock_reserve(&pll_clock, &CYHAL_CLOCK_PLL[0]);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_clock_set_frequency(&pll_clock, AUDIO_SYS_CLOCK_HZ, NULL);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_clock_set_enabled(&pll_clock, true, true);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Initialize the audio subsystem clock (HFCLK1) */
    rslt = cyhal_clock_reserve(&audio_clock, &CYHAL_CLOCK_HF[1]);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_clock_set_source(&audio_clock, &pll_clock);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Drop HFCK1 frequency for power savings */
    cyhal_clock_set_divider(&audio_clock, 4);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    cyhal_clock_set_enabled(&audio_clock, true, true);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Initialize the system clock (HFCLK0) */
    rslt = cyhal_clock_reserve(&system_clock, &CYHAL_CLOCK_HF[0]);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_clock_set_source(&system_clock, &pll_clock);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Initialize the Master Clock with a PWM */
    rslt = cyhal_pwm_init(&mclk_pwm, CYBSP_D0, NULL);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_pwm_set_duty_cycle(&mclk_pwm, MCLK_DUTY_CYCLE, MCLK_FREQ_HZ);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    rslt = cyhal_pwm_start(&mclk_pwm);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Wait for the MCLK to clock the audio codec */
    cyhal_system_delay_ms(1u);

    /* Initialize the I2S */
    rslt = cyhal_i2s_init(&i2s, &i2s_pins, NULL, &i2s_config, &audio_clock);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);
    cyhal_i2s_register_callback(&i2s, i2s_isr_handler, NULL);
    cyhal_i2s_enable_event(&i2s, CYHAL_I2S_ASYNC_TX_COMPLETE,
                                  CYHAL_ISR_PRIORITY_DEFAULT, true);

    /* Initialize the I2C to use with the audio codec */
    cyhal_i2c_init(&i2c, CYBSP_I2C_SDA, CYBSP_I2C_SCL, NULL);
    cyhal_i2c_configure(&i2c, &i2c_config);

    rslt = mtb_ak4954a_init(&i2c);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    /* Enable the AK4954A audio codec with the default volume */
    mtb_ak4954a_activate();
    mtb_ak4954a_adjust_volume(AK4954A_HP_VOLUME_DEFAULT);

    /* Setup complete */

    /* Start the I2S TX */
    cyhal_i2s_start_tx(&i2s);

    /* Initiate a transfer */
    rslt = cyhal_i2s_write_async(&i2s, wave_data, WAVE_SIZE);
    CY_ASSERT(rslt == CY_RSLT_SUCCESS);

    for(;;) { }
}
```


### More information

* [API Reference Guide](https://infineon.github.io/audio-codec-ak4954a/html/index.html)
* [Cypress Semiconductor, an Infineon Technologies Company](http://www.cypress.com)
* [Infineon GitHub](https://github.com/Infineon)
* [ModusToolbox™](https://www.infineon.com/modustoolbox)
* [PSoC™ 6 Code Examples using ModusToolbox™ IDE](https://github.com/Infineon/Code-Examples-for-ModusToolbox™-Software)
* [ModusToolbox™ Software](https://github.com/Infineon/modustoolbox-software)

---
© Cypress Semiconductor Corporation (an Infineon company) or an affiliate of Cypress Semiconductor Corporation, 2019-2024.
