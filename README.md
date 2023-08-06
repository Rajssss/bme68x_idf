# BME 68x ESP-IDF Component 
 
Fully working, read all parameters correctly.
Based on Arduino Lib by Bosch.
## Description  

1. The componenet depends on my modified i2cdev componment for all i2c bus i/o which can be found here: https://github.com/Rajssss/esp_i2cdev_mod
2. Only I2C support for now
3. Kconfig/menuconfig support is WIP, see #define marcos in bme68xLibrary.cpp file to configure the driver for now.
	<pre>  
	#define I2C_SDA_GPIO 			40
	#define I2C_SCL_GPIO 			39
	#define I2C_PORT 			0
	#define I2C_BME68x_ADDR		0x76
	#define I2C_FREQ_HZ 			40000
	</pre>  
4. Tested on ESP-IDF v4.4 (ESP-IDF v4.4-beta1-284-gd83021a6e8-dirty) (release/v4.4 branch)

## Example Code for IDF  

    #include <stdio.h>
    #include <stdint.h>
    #include <string.h>
    #include "bme68xLibrary.h"
    #include "i2cdev.h"
    #include "driver/gpio.h"
    #include "esp_log.h"
    #include "sdkconfig.h"
    
    #define NOP() asm volatile ("nop")
   
    static const char* TAG = "main";
    
    Bme68x bme;
    i2c_dev_t dev;
    
    
    static unsigned long IRAM_ATTR micros()
    {
        return (unsigned long) (esp_timer_get_time());
    }
    
    static void IRAM_ATTR delayMicroseconds(uint32_t us)
    {
        uint32_t m = micros();
        if(us){
            uint32_t e = (m + us);
            if(m > e){ //overflow
                while(micros() > e){
                    NOP();
                }
            }
            while(micros() < e){
                NOP();
            }
        }
    }
    
    
    extern "C" void app_main()
    {    
    	i2cdev_init(); // from i2cdev
    
    	/* initializes the sensor based on SPI library */
    	bme.begin(&dev);
    
    	ESP_LOGI(TAG,"checkStatus: %d", bme.checkStatus());
    
    	if(bme.checkStatus())
    	{
    		ESP_LOGI(TAG,"statusString: %s", bme.statusString());
    
    		if (bme.checkStatus() == BME68X_ERROR)
    		{
    			ESP_LOGE(TAG,"Sensor error: %s", bme.statusString());
    			return;
    		}
    		else if (bme.checkStatus() == BME68X_WARNING)
    		{
    			ESP_LOGW(TAG,"Sensor Warning: %s", bme.statusString());
    		}
    	}
    
    	/* Set the default configuration for temperature, pressure and humidity */
    	bme.setTPH();
    
    	/* Set the heater configuration to 300 deg C for 100ms for Forced mode */
    	bme.setHeaterProf(300, 100);
    
    	while(1)
    	{
    		bme68xData data;
    
    		bme.setOpMode(BME68X_FORCED_MODE);
    		delayMicroseconds(bme.getMeasDur());
    
    		if (bme.fetchData())
    		{
    			bme.getData(data);
    
    			ESP_LOGI(TAG,"Temp = %f\n", data.temperature);
    
    			ESP_LOGI(TAG,"Pressure = %f\n", data.pressure);
    
    			ESP_LOGI(TAG,"Humidity = %f\n", data.humidity);
    
    			ESP_LOGI(TAG,"GAS RES = %f\n", data.gas_resistance);
    		}
    	}
    }


