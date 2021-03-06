# easyq-lwip-client

C Implementation of EasyQ client using LWIP library, suitable for esp8266 module

## How to use

First, install the (esp-open-rtos)[https://github.com/SuperHouse/esp-open-rtos]

The include the headers:

```C

/* Very basic example that just demonstrates we can run at all!
 */
#include "espressif/esp_common.h"
#include "esp/uart.h"
#include "FreeRTOS.h"
#include "task.h"

#include "ssid_config.h"
#include "easyq_config.h"

#include "easyq.h"
#include "common.h" // wait_for_wifi_connection() and delay()


#define STATUS_QUEUE "status"
#define COMMAND_QUEUE "cmd"

EQSession * eq;


void sender(void* args) {
    err_t err;
    int c = 0;
    char buff[16];
    Queue * queue = Queue_new(STATUS_QUEUE);
    while (1) {
        delay(1000);

    	// Wait for wifi conection
        wait_for_wifi_connection();
        
        printf("Inititlizing easyq\n");
        err = easyq_init(&eq);
        if (err != ERR_OK) {
            printf("Cannot Inititalize the easyq\n");
            continue;
        }
        printf("Session ID: %s\n", eq->id);
        while (1) {
            sprintf(buff, "%08d", c);
            easyq_push(eq, queue, buff, -1);
            delay(5000);
            c++;
        }
    }
}


void listener(void* args) {
    err_t err;
    char * buff;
    char *queue_name;
    size_t buff_len;
    Queue * queue = Queue_new(COMMAND_QUEUE);
    while (1) {
        delay(1000);
        if (eq == NULL || !eq->ready) {
            continue;
        }
        printf("EQ Ready: %d\n", eq->ready);
        err = easyq_pull(eq, queue);
        if (err != ERR_OK) {
            continue;
        }
        while(1) {
            err = easyq_read_message(eq, &buff, &queue_name, &buff_len);
            if (err != ERR_OK) {
                printf("Error reading from EasyQ");
                continue;
            }
            if (buff_len <= 0) {
                continue;
            }
            printf("MESSAGE: %s FROM QUEUE: %s LEN: %d\n", buff, queue_name, buff_len);
        }
    }
}


void user_init(void)
{
    uart_set_baud(0, 115200);
    printf("SDK version:%s\n", sdk_system_get_sdk_version());

    // Wifi
    struct sdk_station_config config = {
        .ssid     = WIFI_SSID,
        .password = WIFI_PASS,
    };

    /* required to call wifi_set_opmode before station_set_config */
    sdk_wifi_set_opmode(STATION_MODE);
    sdk_wifi_station_set_config(&config);

    xTaskCreate(sender, "sender", 512, NULL, 2, NULL);
    xTaskCreate(listener, "listener", 384, NULL, 2, NULL);
}

```

And include this in your projects make file:

```make
PROGRAM = simple
EXTRA_COMPONENTS = path/to/easyq-lwip-client/
include path/to/esp-open-rtos/common.mk
```

Export path to SDK and output bin dir.
```bash
export SDK_PATH=path/to/esp-open-rtos
export BIN_PATH=path/to/out/binaries
```

Then

```bash
make [flash]
```

See the `examples` directory for more information.

