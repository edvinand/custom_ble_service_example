# Custom Service Tutorial

This tutorial will show you how to create a custom service with a custom value characteristic in the ble_app_template project found in the Nordic nRF5 SDK v15.3.0. This tutorial can be seen as the combined version of the BLE Advertising / Services / Characteristics , A Beginner's Tutorial series, which I strongly recommend to take a look at as they go deeper into the matter than this tutorial. Note, these tutorials are compatible with an older SDK version, but the theory regarding Bluetooth Low Energy has not changed much.

The aim of this tutorial is simply to create one service with one characteristic without too much theory in between the steps. There are no .c or .h files that needs to be downloaded as we will be starting from scratch in the ble_app_template project.

However, if you simply want to compile the example without doing the tutorial steps then you can be clone this repo into SDK v15.3.0/examples/ble_peripheral.

# HW Requirements
- RF52 Development Kit

# SW Requirements
- nRF5 SDKv15.3.0 [(download page)](https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK/Download#infotabs)
-Latest version of Segger Embedded Studio [(download page)](https://www.segger.com/downloads/embedded-studio/) or latest version of Keil ARM MDK [(download page)](https://www.keil.com/demo/eval/arm.htm) or the GCC ARM Embedded 7 2018-q2-update toolchain [(download page)](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads)
- nRF Connect for Mobile [(download page)](https://www.nordicsemi.com/Software-and-Tools/Development-Tools/nRF-Connect-for-mobile)
- nRF Command Line Tools [(download page)](https://www.nordicsemi.com/Software-and-Tools/Development-Tools/nRF-Command-Line-Tools)

# IDE/Toolchain Support
Nordic Semiconductor added Segger Embedded Studio support in SDKv14.1.0 and the tutorial has been written with that IDE in mind, i.e. steps to change Memory Settings/build Parameters will mainly be for SES. However, the code should compile with all the IDEs/toolchains in the list:
- Segger Embedded Studio
- Make and GCC
- Keil

# Tutorial Steps
**Step 1 - Getting started**
1. Download nRF5_SDK_15.3.0_59ac345 from the [download page](https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK/Download#infotabs) and extract the zip to your drive, e.g. C:\NordicSemi\nRF5_SDK_15.3.0_59ac345.
2. Navigate to the nRF5_SDK_15.3.0_59ac345/examples/ble_peripheral folder and find the ble_app_template project folder.
3. Create a copy of the folder and name it `custom_ble_service_example`.
4. Navigate to custom_ble_service_example\pca10040\s132\ses and open the `ble_app_template_pca10040_s132.emProject` project.

**Step 2 - Creating a Custom Base UUID**
The first thing we need to do is to create a new .c file, let call it ble_cus.c (**Cu**stom **S**ervice), and its accompaning .h file, ble_cus.h. Create the two files in the same folder as the main.c file. At the top of the header file ble_cus.h we'll need to include the following .h files:
```C
/* This code belongs in ble_cus.h */
#include <stdint.h>
#include <stdbool.h>
#include "ble.h"
#include "ble_srv_common.h"
```
Next, we're going to need a 128-bit UUID for our custom service since we're not going to implement our service with one of the 16-bit Bluetooth SIG UUIDs that are reserved for standardized profiles. There are several ways to generate a 128-bit UUID, but we'll use [this](https://www.uuidgenerator.net/version4) Online UUID generator. The webpage will generate a random 128-bit UUID, which in my case was
```
dd8c03aa-e23a-425c-b8be-96721cd710fe
```
The UUID is given as sixteen octets of a UUID are represented as 32 hexadecimal (base16) digits, displayed in five groups seperated by hyphens, in the form of 8-4-4-4-12. The 16 octets are given in big-endian, while we use the small-endian representation in our SDK. Thus we must reverse the byte-ordering when we define our UUID base in the ble_cus.h, as shown below.
```C
/* This code belongs in ble_cus.h */
#define CUSTOM_SERVICE_UUID_BASE          {0xFE, 0x10, 0xD7, 0x1C, 0x72, 0x96, 0xBE, 0xB8, \
                                          0x42, 0x5C, 0xE2, 0x3A, 0xAA, 0x03, 0x8C, 0xDD}
```
Now that we have defined our Base UUID, we need to define a 16-bit UUID for the Custom Service and a 16-bit UUID for a Custom Value Characteristic.
```C
/* This code belongs in ble_cus.h */
#define CUSTOM_SERVICE_UUID               0x1400
#define CUSTOM_VALUE_CHAR_UUID            0x1401
```
The values for the 16-bit UUIDs that will be inserted into the base UUID can be coosen by random.
**Step 3 - Implementing the Custom Service**
First things first, we need to include the ble_cus.h header file we just created as well as some common SDK header files in ble_cus.c.
```C
/* This code belongs in ble_cus.c */
#include "sdk_common.h"
#include "ble_srv_common.h"
#include "ble_cus.h"
#include <string.h>
#include "nrf_gpio.h"
#include "boards.h"
#include "nrf_log.h"
```
The next step is  to add a macro for defining a Custom Service (ble_cus) by adding the following snippet below the includes in ble_cus.h
```C
/* This code belongs in ble_cus.h */
/**@brief   Macro for defining a ble_cus instance.
 *
 * @param   _name Name of the instance.
 * @hideinitializer
 */
#define BLE_CUS_DEF(_name)
static ble_cus_t _name;
```
*Note that the project does not compile at this point in time*<br/>
We will use this macro to define a custom service instance in main.c later in the tutorial.
Ok, so far so good. Now we need to create two structures in ble_cus.h, one Custom Service init structure, ble_cus_init_t struct to hold all the options and data needed to initialize our custom service.
```C
/* This code belongs in ble_cus.h*/

/**@brief Custom Service init structure. This contains all options and data needed for
 *        initialization of the service.*/
typedef struct
{
    uint8_t                       initial_custom_value;           /**< Initial custom value */
    ble_srv_cccd_security_mode_t  custom_value_char_attr_md;      /**< Initial security level for Custom characteristics attribute */
} ble_cus_init_t;
```
*Note that the project still doesn't compile*<br/>
The second struct that we need to create is the Custom Service structure, ble_cus_s, which holds the status information of the service.
```C
/* This code belongs in ble_cus.h*/

/**@brief Custom Service structure. This contains various status information for the service. */
struct ble_cus_s
{
    uint16_t                      service_handle;                 /**< Handle of Custom Service (as provided by the BLE stack). */
    ble_gatts_char_handles_t      custom_value_handles;           /**< Handles related to the Custom Value characteristic. */
    uint16_t                      conn_handle;                    /**< Handle of the current connection (as provided by the BLE stack, is BLE_CONN_HANDLE_INVALID if not in a connection). */
    uint8_t                       uuid_type; 
};
```
*still doesn't compile.*<br/>
The next step is to add a forward declaration of the ble_cus_t type. You need to insert this code *above* the `BLE_CUS_DEF` define
```C
/* This code belongs in ble_cus.h*/

// Forward declaration of the ble_cus_t type.
typedef struct ble_cus_s ble_cus_t;
```
*Now the project should compile again*<br/>
The first function we're going to implement is the ble_cus_init function, which we're going to initialize our service with. First, we need to do (not done)
