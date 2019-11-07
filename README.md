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
### Step 1 - Getting started
1. Download nRF5_SDK_15.3.0_59ac345 from the [download page](https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK/Download#infotabs) and extract the zip to your drive, e.g. C:\NordicSemi\nRF5_SDK_15.3.0_59ac345.
2. Navigate to the nRF5_SDK_15.3.0_59ac345/examples/ble_peripheral folder and find the ble_app_template project folder.
3. Create a copy of the folder and name it `custom_ble_service_example`.
4. Navigate to custom_ble_service_example\pca10040\s132\ses and open the `ble_app_template_pca10040_s132.emProject` project.

### Step 2 - Creating a Custom Base UUID
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
### Step 3 - Implementing the Custom Service
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
#define BLE_CUS_DEF(_name)                                                                          \
static ble_cus_t _name;                                                                             \
```
*Note1: The project does not compile at this point in time</br>
Note2: Remember the `\` at the end of the line*</br>
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
The first function we're going to implement is the ble_cus_init function, which we're going to initialize our service with. The first thing we need to do is to add its function declaration in the ble_cus.h file.
```C
/* This code belongs in ble_cus.h */

/**@brief Function for initializing the Custom Service.
 *
 * @param[out]  p_cus       Custom Service structure. This structure will have to be supplied by
 *                          the application. It will be initialized by this function, and will later
 *                          be used to identify this particular service instance.
 * @param[in]   p_cus_init  Information needed to initialize the service.
 *
 * @return      NRF_SUCCESS on successful initialization of service, otherwise an error code.
 */
ret_code_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init);
```
The first thing we should do upon entering ble_cus_init is to check that none of the pointers that we passed as arguments are NULL and declare the two variables err_code and ble_uuid.
```C
/* This code belongs in ble_cus.c */
uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    ret_code_t err_code;
    ble_uuid_t ble_uuid;
}
```
After verifying that the pointers are valid we can initialize the Custom Service structure
```C
/* This code belongs in ble_cus_init() in ble_cus.c */

// Initialize service structure
p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;
```
This consists of setting the connection handle to invalid (should only be valid when we're in a connection). Next, we're going to add our custom (also referred to as vendor specific) base UUID to the BLE stack's table.
```C
/* This code belongs in ble_cus_init() ble_cus.c*/

// Add Custom Service UUID
ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
err_code =  sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
VERIFY_SUCCESS(err_code);

ble_uuid.type = p_cus->uuid_type;
ble_uuid.uuid = CUSTOM_SERVICE_UUID;
```
We're almost done, the last thing we have to do is to add the Custom Service declaration to the BLE Stack's GATT table.
```C
/* This code belongs in ble_cus_init() in ble_cus.c */

// Add the Custom Service
err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
if (err_code != NRF_SUCCESS)
{
    return err_code;
}
```
If you have followed the steps correctly, then the ble_cus_init should look like this:
```C
/* This code belongs in ble_cus.c */

uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    ret_code_t  err_code;
    ble_uuid_t  ble_uuid;

    // Initialize service structure
    p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;

    // Add Custom Service UUID
    ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
    err_code =  sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
    VERIFY_SUCCESS(err_code);
    
    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_SERVICE_UUID;

    // Add the Custom Service
    err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }
    return err_code;
}
```
### Step 4 - Initializing the Service and adavertising our 128-bit UUID.
Now it's time to initialize the service in main.c and put our 128-bit UUID in the advertisement packet so that other BLE devices can see that our device has the Custom Service.</br>
First, add the ble_cus,h file to the include list in main.c.
```C
/* This code belongs in main.c */
#include "ble_cus.h"
```
The next step is to find the empty services_init function in main.c which should look like this
```C
*/

/**@brief Function for initializing services that will be used by the application.
 */
static void services_init(void)
{
    ret_code_t         err_code;
    nrf_ble_qwr_init_t qwr_init = {0};

    // Initialize Queued Write Module.
    qwr_init.error_handler = nrf_qwr_error_handler;

    err_code = nrf_ble_qwr_init(&m_qwr, &qwr_init);
    APP_ERROR_CHECK(err_code);

    /* YOUR_JOB: Add code to initialize the services used by the application.
       ble_xxs_init_t                     xxs_init;
       ble_yys_init_t                     yys_init;

       // Initialize XXX Service.
       memset(&xxs_init, 0, sizeof(xxs_init));

       xxs_init.evt_handler                = NULL;
       xxs_init.is_xxx_notify_supported    = true;
       xxs_init.ble_xx_initial_value.level = 100;

       err_code = ble_bas_init(&m_xxs, &xxs_init);
       APP_ERROR_CHECK(err_code);

       // Initialize YYY Service.
       memset(&yys_init, 0, sizeof(yys_init));
       yys_init.evt_handler                  = on_yys_evt;
       yys_init.ble_yy_initial_value.counter = 0;

       err_code = ble_yy_service_init(&yys_init, &yy_init);
       APP_ERROR_CHECK(err_code);
     */
}
```
Ok, we're going to do as we're told, i.e. create a ble_cus_init_t struct and populate it with the necessary data and then pass it as an argument to our service init function ble_cus_init();
```C
/* This code belongs in main.c */
/**@brief Function for initializing services that will be used by the application.
 */
static void services_init(void)
{
    ret_code_t         err_code;
    nrf_ble_qwr_init_t qwr_init = {0};
    
    // Initialize Queued Write Module.
    qwr_init.error_handler = nrf_qwr_error_handler;
    
    err_code = nrf_ble_qwr_init(&m_qwr, &qwr_init);
    APP_ERROR_CHECK(err_code);
    
    /* YOUR_JOB: Add code to initialize the services used by the application.*/
    ble_cus_init_t                     cus_init;
    
    
    // Initialize CUS Service.
    memset(&cus_init, 0, sizeof(cus_init));
    
    err_code = ble_cus_init(&m_cus, &cus_init);
    APP_ERROR_CHECK(err_code);     
}
```
Now that we have initialized the service we have to add the 128-bit UUID to the advertisement packet. If you navigate to the top of main.c you should find the m_adv_uuids array.
```C
// YOUR_JOB: Use UUIDs for service(s) used in your application.
static ble_uuid_t m_adv_uuids[] =                                               /**< Universally unique service identifiers. */
{
    {BLE_UUID_DEVICE_INFORMATION_SERVICE, BLE_UUID_TYPE_BLE}
};
```
We need to replace the BLE_UUID_DEVICE_INFORMATION_SERVICE with the CUSTOM_SERVICE_UUID we defined in ble_cus.h as well as replace BLE_UUID_TYPE_BLE with BLE_UUID_TYPE_VENDOR_BEGIN since this is a 128-bit vendor specific UUID and not a 16-bit Bluetooth SIG UUID. m_adv_uuids should now look like this
```C
// YOUR_JOB: Use UUIDs for service(s) used in your application.
static ble_uuid_t m_adv_uuids[] =                                               /**< Universally unique service identifiers. */
{
    {CUSTOM_SERVICE_UUID, BLE_UUID_TYPE_VENDOR_BEGIN}
};
```
After this step we need to tell the BLE stack that we're using a vendor-specific 128-bit UUID and not a 16-bit UUID. This is done by changing the following define in sdk_config.h from
```C
#define NRF_SDH_BLE_VS_UUID_COUNT 0
```
to
```C
#define NRF_SDH_BLE_VS_UUID_COUNT 1
```
Now, adding a vendor-specific UUID to the BLE stack results in the RAM requirement of the SoftDevice increasing, which we need to take into account.</br>
**GCC:** If you're compiling the code using armgcc then you need to open the ble_app_template_gcc_nrf52.ld file in the nRF5_SDK\nRF5_nRF5_SDK_15.3.0_59ac345\examples\ble_peripheral\custom_ble_service_example\pca10040\s132\armgcc folder and modify the memory section as shown below.
```
MEMORY
{
  FLASH (rx) : ORIGIN = 0x26000, LENGTH = 0x5a000
  RAM (rwx) : ORIGIN = 0x20002220, LENGTH = 0xdde0
}
```
**Segger Embedded Studio(SES):** Click "Project -> Edit Options", select the Common Configuration, then select Linker and then open the Section Placement Macros Section and modify RAM_START IRAM1 to 0x20002220 and RAM_SIZE to 0xDDE0, as shown in the screenshot below
Memory Settings Segger Embedded Studio | 
------------ |
<img src="https://github.com/bjornspockeli/custom_ble_service_example/blob/master/images/memory_settings_SDK_v15_SES.JPG" width="1000"> |

**Keil:** Click "Options for Target" in Keil and modify the Read/Write Memory Areas so that IRAM1 has the start address 0x20002220 and size 0xDDE0, as shown in the screenshot below

Memory Settings Keil | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/memory_settings_SDK_v15_3_0_Keil.JPG" width="1000"> |
