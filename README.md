# Custom Service Tutorial

Please note that this is just an updated version of @bjornspockeli's tutorial, adjusted to work with the SDK15.3.0. The previous version is found [here](https://github.com/bjornspockeli/custom_ble_service_example) 
</br></br>

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
The first thing we need to do is to create a new .c file, let call it ble_cus.c (**Cu**stom **S**ervice), and its accompaning .h file, ble_cus.h. Create the two files **in the same folder as the main.c file**. In the top of the header file ble_cus.h we'll need to include the following .h files:
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
Ok, so far so good. Now we need to create two structures in ble_cus.h, one Custom Service init structure, ble_cus_init_t struct to hold all the options and data needed to initialize our custom service. </br>
Define your instance of m_cus near the top of main.c.
```C
/* This code belongs to main.c */
BLE_CUS_DEF(m_cus);
```
*Note: When you use this macro in main.c with the input parameter `m_cus`, it means that you create a static ble_cus_t with the name "m_cus", which we can use later in main.c.
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
ret_code_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
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

ret_code_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
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
The last if check is in fact redundant, but we will change this function later, adding more calls.
### Step 4 - Initializing the Service and adavertising our 128-bit UUID.
Now it's time to initialize the service in main.c and put our 128-bit UUID in the advertisement packet so that other BLE devices can see that our device has the Custom Service.</br>
First, add the ble_cus,h file to the include list in main.c.
```C
/* This code belongs in main.c */
#include "ble_cus.h"
```
The next step is to find the empty services_init function in main.c which should look like this
```C
/*
/**@brief Function for initializing services that will be used by the application.
 *
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
**Segger Embedded Studio(SES):** Click "Project -> Edit Options", select the Common Configuration, then select Linker and then open the Section Placement Macros Section and modify RAM_START to 0x20002220 and RAM_SIZE to 0xDDE0, as shown in the screenshot below
Memory Settings Segger Embedded Studio | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/memory_settings_SDK_v15_3_0_SES.jpg" width="1000"> |

**Keil:** Click "Options for Target" in Keil and modify the Read/Write Memory Areas so that IRAM1 has the start address 0x20002220 and size 0xDDE0, as shown in the screenshot below

Memory Settings Keil | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/memory_settings_SDK_v15_3_0_Keil.jpg" width="1000"> |

The final step we have to do is to change the calling order in main() so that services_init() is called before advertising_init().
This is because we need to add the CUSTOM_SERVICE_UUID_BASE to the BLE stack's table using sd_ble_uuid_vs_add() in ble_cus_init() before we call advertising_init(). Doing it the other way around will cause advertising_init() to return an error code.</br>
That should be it. Compile the ble_app_template project, flash the S132 v6.1.1 SoftDevice *(If you use segger embedded Studio this is done automatically by the IDE)* and then flash the ble_app_template application. LED1 on your nRF52DK should now start blinking, indicating that it is advertising. Use nRF Connect for Android/iOS to scan for the device and view the content of the advertisement package. If you connect to the device you should see the service listed as an "Unknown Service" since we're using a vendor-specific UUID.

Advertising Device  |  Service listed in the GATT table    |
------------ | ------------- | 
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_scanning.png" width="250">  | <img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_adv_service.jpg" width="250"> |

### Step 5 - Adding a custom Value Characteristic to the Custom Service.
A Service is nothing without a characteristic, so let's add one of those by creating the custom_value_char_add function to ble_cus.c.</br>
The first thing we have to do is to declare the function and then add several metadata variables that we will later populate, as shown in the snippet below.
```C
/* This code belongs in ble_cus.c */

/**@brief Function for adding the Custom Value characteristic.
 *
 * @param[in]   p_cus        Custom Service structure.
 * @param[in]   p_cus_init   Information needed to initialize the service.
 *
 * @return      NRF_SUCCESS on success, otherwise an error code.
 */
static ret_code_t custom_value_char_add(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    ret_code_t          err_code;
    ble_gatts_char_md_t char_md;
    ble_gatts_attr_md_t cccd_md;
    ble_gatts_attr_t    attr_char_value;
    ble_uuid_t          ble_uuid;
    ble_gatts_attr_md_t attr_md;
}
```
Now starts the rather tedious part of populating all these variables. We'll start with char_md, which sets the properties that will be displayed to the central during cervice discovery.
```C
/* This code belongs in custom_value_char_add() in ble_cus.c*/

    memset(&char_md, 0, sizeof(char_md));

    char_md.char_props.read   = 1;
    char_md.char_props.write  = 1;
    char_md.char_props.notify = 0; 
    char_md.p_char_user_desc  = NULL;
    char_md.p_char_pf         = NULL;
    char_md.p_user_desc_md    = NULL;
    char_md.p_cccd_md         = NULL; 
    char_md.p_sccd_md         = NULL;
}
```
So we want to be able to both write and read to our Custom Value characteristic, but we do not want to enable the notify property until later. Next we're going to populate the attr_md, which actually sets the properties (i.e. accessibility of the attribute).
```C
    /* This code belongs in custom_value_char_add() in ble_cus.c*/

    memset(&attr_md, 0, sizeof(attr_md));

    attr_md.read_perm  = p_cus_init->custom_value_char_attr_md.read_perm;
    attr_md.write_perm = p_cus_init->custom_value_char_attr_md.write_perm;
    attr_md.vloc       = BLE_GATTS_VLOC_STACK;
    attr_md.rd_auth    = 0;
    attr_md.wr_auth    = 0;
    attr_md.vlen       = 0;
}
```

The permissions set in the attr_md struct should correspond with the properties set in the characteristic metadata struct char_md. We're going to provide the permissions in the Custom Service init structure that we pass to ble_cus_init in services_init(). The .vloc option is set to BLE_GATTS_VLOCK_STACK as we want the characteristic to be stored in the SoftDevice RAM section, and not in the Application RAM section.</br>
The next variable that we have to populate is the ble_uuid, which is going to hold our CUSTOM_VALUE_CHAR_UUID and is of the same type as the CUSTOM_SERVICE_UUID_BASE, i.e. vendor specific, which we specified in the .uuid_type field of Custom Service structure when we added the CUSTOM_SERVICE_UUID_BASE to the BLE stack's table.
```C
    /* This code belongs in custom_value_char_add() in ble_cus.c*/

    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_VALUE_CHAR_UUID;
```

Next, we're going to populate the attr_char_value struct, which sets the UUID, which points to the attribute metadata and sets the size of the characteristic, in our case a single byte (uint8_t).
```C
    /* This code belongs in custom_value_char_add() in ble_cus.c*/

    memset(&attr_char_value, 0, sizeof(attr_char_value));

    attr_char_value.p_uuid    = &ble_uuid;
    attr_char_value.p_attr_md = &attr_md;
    attr_char_value.init_len  = sizeof(uint8_t);
    attr_char_value.init_offs = 0;
    attr_char_value.max_len   = sizeof(uint8_t);
```

Finally, we're done populating structs and we can add our characteristic by calling sd_ble_gatts_characteristic_add() with the structs as arguments.
```C
/* This code belongs in custom_value_char_add() in ble_cus.c*/

    err_code = sd_ble_gatts_characteristic_add(p_cus->service_handle, &char_md,
                                               &attr_char_value,
                                               &p_cus->custom_value_handles);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    return NRF_SUCCESS;
```

After all that hard work (copy-pasting) your custom_value_char_add() functino should look like this.
```C
/* This code belongs in ble_cus.c*/

static ret_code_t custom_value_char_add(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    ret_code_t          err_code;
    ble_gatts_char_md_t char_md;
    ble_gatts_attr_md_t cccd_md;
    ble_gatts_attr_t    attr_char_value;
    ble_uuid_t          ble_uuid;
    ble_gatts_attr_md_t attr_md;

    memset(&char_md, 0, sizeof(char_md));

    char_md.char_props.read   = 1;
    char_md.char_props.write  = 1;
    char_md.char_props.notify = 0; 
    char_md.p_char_user_desc  = NULL;
    char_md.p_char_pf         = NULL;
    char_md.p_user_desc_md    = NULL;
    char_md.p_cccd_md         = NULL; 
    char_md.p_sccd_md         = NULL;
    
    memset(&attr_md, 0, sizeof(attr_md));

    attr_md.read_perm  = p_cus_init->custom_value_char_attr_md.read_perm;
    attr_md.write_perm = p_cus_init->custom_value_char_attr_md.write_perm;
    attr_md.vloc       = BLE_GATTS_VLOC_STACK;
    attr_md.rd_auth    = 0;
    attr_md.wr_auth    = 0;
    attr_md.vlen       = 0;

    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_VALUE_CHAR_UUID;

    memset(&attr_char_value, 0, sizeof(attr_char_value));

    attr_char_value.p_uuid    = &ble_uuid;
    attr_char_value.p_attr_md = &attr_md;
    attr_char_value.init_len  = sizeof(uint8_t);
    attr_char_value.init_offs = 0;
    attr_char_value.max_len   = sizeof(uint8_t);

    err_code = sd_ble_gatts_characteristic_add(p_cus->service_handle, &char_md,
                                               &attr_char_value,
                                               &p_cus->custom_value_handles);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    return NRF_SUCCESS;
}
```

The final step is to call custom_value_char_add() at the end of ble_cus_init the service has been added, i.e.
```C
ret_code_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    ret_code_t err_code;
    ble_uuid_t ble_uuid;

    // Initialize the service structure
    p_cus->conn_handle                    = BLE_CONN_HANDLE_INVALID;

    // Add Custom Service UUID
    ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
    err_code = sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
    VERIFY_SUCCESS(err_code);

    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_SERVICE_UUID;

    err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }
    // Add Custom Value charracteristic
    return custom_value_char_add(p_cus, p_cus_init);
}
```

Compile the project and flash it to your nRF52832 DK. If you open the nRF Connect app on your smartphone, scan and connect to the device, you should see that the characteristic has been added by clicking on the service, as shown in the screenshot below.

Service and Characteristic | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_connected_service_char.jpg" width="250"> |

### Step 6 - Handling events from the SoftDevice.
Great, we now have a Custom Service and a Custom Value Characteristic, but we want to be able to write to the characteristic and perform a specific task based on the value that was written to the characteristic, e.g. turn on a LED. However, before we can do that we need to do some event handling in ble_cus.h and ble_cus.c.</br>
Lastly, we're going to add the ble_cus_on_ble_evt function declaration to ble_cus.h, which will handle the events of the ble_cus_evt_type_t from our service.

```C
/* This code belongs in ble_cus.h*/

/**@brief Function for handling the Application's BLE Stack events.
 *
 * @details Handles all events from the BLE stack of interest to the Battery Service.
 *
 * @note 
 *
 * @param[in]   p_ble_evt  Event received from the BLE stack.
 * @param[in]   p_context  Custom Service structure.
 */
void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context);
```

We're now going to implement the ble_cus_on_ble_evt event handler in ble_cus.c. Upon entry it's considered as good practice to check that none of the pointers that we provide as arguments are NULL.
```C
/* This code belongs in ble_cus.c*/

void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context)
{
    ble_cus_t * p_cus = (ble_cus_t *) p_context;
    
    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
}
```
After the NULL check we're going to add a switch-case to check which event that has been propagated to the application by the softdevice.
```C
/* This code belongs in ble_cus.c*/

void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context)
{
    ble_cus_t * p_cus = (ble_cus_t *) p_context;
    
    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
    
    switch (p_ble_evt->header.evt_id)
    {
        case BLE_GAP_EVT_CONNECTED:
            break;

        case BLE_GAP_EVT_DISCONNECTED:
            break;

        default:
            // No implementation needed.
            break;
    }
}
```

For now we only need to care about the BLE_GAP_EVT_CONNECTED and BLE_GAP_EVT_DISCONNECTED events. So let's create the two functions on_connect() and on_disconnect(), starting with on_connect(). The only thing we need to do when we get the Connect event is to assign the connection handle in the Custom Service structure to the connection handle that is passed with the evemt.

```C
/* This code belongs in ble_cus.c */

/**@brief Function for handling the Connect event.
 *
 * @param[in]   p_cus       Custom Service structure.
 * @param[in]   p_ble_evt   Event received from the BLE stack.
 */
static void on_connect(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    p_cus->conn_handle = p_ble_evt->evt.gap_evt.conn_handle;
}
```
Similarly, when we get the Disconnect event, the only thing we need to do is invalidate the connection handle in the Custom Servicee structure since the connection is now dead.

```C
/* This code belongs in ble_cus.c */

/**@brief Function for handling the Disconnect event.
 *
 * @param[in]   p_cus       Custom Service structure.
 * @param[in]   p_ble_evt   Event received from the BLE stack.
 */
static void on_disconnect(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    UNUSED_PARAMETER(p_ble_evt);
    p_cus->conn_handle = BLE_CONN_HANDLE_INVALID;
}
```

Now that we have one function for each event we only need to call the function in ble_cus_on_ble_evt, i.e.
```C
/* This code belongs in ble_cus.c*/

void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context)
{
    ble_cus_t * p_cus = (ble_cus_t *) p_context;

    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
    
    switch (p_ble_evt->header.evt_id)
    {
        case BLE_GAP_EVT_CONNECTED:
            on_connect(p_cus, p_ble_evt);
            break;

        case BLE_GAP_EVT_DISCONNECTED:
            on_disconnect(p_cus, p_ble_evt);
            break;

        default:
            // No implementation needed.
            break;
    }
}
```
The last thing we have to do is to make sure that our ble_cus_on_ble_evt event handler function receives SoftDevice events. This is done registering the ble_cus_on_ble_evt event handler as event observer using the NRF_SDH_BLE_OBSERVER() macro. It is convenient to do this within the BLE_CUS_DEF macro that we defined in ble_cus.h, which should now be modified as shown below

```C
/* This code belongs in ble_cus.h*/

#define BLE_CUS_DEF(_name)                                                                          \
static ble_cus_t _name;                                                                             \
NRF_SDH_BLE_OBSERVER(_name ## _obs,                                                                 \
                     BLE_HRS_BLE_OBSERVER_PRIO,                                                     \
                     ble_cus_on_ble_evt, &_name)
```

Compile your project and verify that there are no errors before you proceed to the next step.

### Step 7 - Handling the Write event from the SoftDevice.
Ok, now we really want to be able to write to the characteristic and perform a specific task based on the value that was written to the characteristic, e.g. turn on a LED. How are we going to do that? You guessed it! More event handling!</br>
Whenever a characteristic is written to, a BLE_GATTS_EVT_WRITE event will be propagated to the application and dispatched to the functions registered in the NRF_SDH_BLE_OBSERVER. So this means that we need to add another case to our ble_cus_on_ble_evt switch-case statement, namely BLE_GATTS_EVT_WRITE
```C
/* This code belongs in ble_cus.c*/

void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context)
{
   ble_cus_t * p_cus = (ble_cus_t *) p_context;

   if (p_cus == NULL || p_ble_evt == NULL)
   {
       return;
   }
   
   switch (p_ble_evt->header.evt_id)
   {
       case BLE_GAP_EVT_CONNECTED:
           on_connect(p_cus, p_ble_evt);
           break;

       case BLE_GAP_EVT_DISCONNECTED:
           on_disconnect(p_cus, p_ble_evt);
           break;
       case BLE_GATTS_EVT_WRITE:
           break;
       default:
           // No implementation needed.
           break;
   }
}
```
Just like we did for the BLE_GAP_EVT_CONNECTED and BLE_GAP_EVT_CONNECTED we're going to create a on_write function that should be called when we get the Write event.
```C
/* This code belongs in ble_cus.c*/

/**@brief Function for handling the Write event.
 *
 * @param[in]   p_cus       Custom Service structure.
 * @param[in]   p_ble_evt   Event received from the BLE stack.
 */
static void on_write(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{

}
```

Now, once we get the Write event we have to get hold of the Write event parameters that are passed with the event and we have to verify that the handle that is written to matches the Custom Value Characteristic handle, i.e.
```C
/* This code belongs in ble_cus.c*/

static void on_write(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    ble_gatts_evt_write_t * p_evt_write = &p_ble_evt->evt.gatts_evt.params.write;
    
    // Check if the handle passed with the event matches the Custom Value Characteristic handle.
    if (p_evt_write->handle == p_cus->custom_value_handles.value_handle)
    {
        // Put specific task here. 
    }
}
```

So lets say that our specific task is to toggle a LED on the nRF5x DK every time the Custom Value Characteristic is written to. We can do this by calling nrf_gpio_pin_toggle on one of the pins connected to the nRF5x DK LEDs, e.g. LED4. In order to do this we'll have to include boards.h and nrf_gpio.h in ble_cus.h as well as call nrf_gpio_pin_toggle in the on_write_function
```C
/* This code belongs in ble_cus.c*/

static void on_write(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    ble_gatts_evt_write_t * p_evt_write = &p_ble_evt->evt.gatts_evt.params.write;
    
    // Check if the handle passed with the event matches the Custom Value Characteristic handle.
    if (p_evt_write->handle == p_cus->custom_value_handles.value_handle)
    {
        nrf_gpio_pin_toggle(LED_4); 
    }
}
```

We've now implemented the necessary event handling so on_write() should be added to the ble_cus_on_ble_evt() function under the BLE_GATTS_EVT_WRITE case, i.e.
```C
/* This code belongs in ble_cus.c*/

void ble_cus_on_ble_evt( ble_evt_t const * p_ble_evt, void * p_context)
{
   ble_cus_t * p_cus = (ble_cus_t *) p_context;

   if (p_cus == NULL || p_ble_evt == NULL)
   {
       return;
   }
   
   switch (p_ble_evt->header.evt_id)
   {
       case BLE_GAP_EVT_CONNECTED:
           on_connect(p_cus, p_ble_evt);
           break;

       case BLE_GAP_EVT_DISCONNECTED:
           on_disconnect(p_cus, p_ble_evt);
           break;
       case BLE_GATTS_EVT_WRITE:
           on_write(p_cus, p_ble_evt);
           break;
       default:
           // No implementation needed.
           break;
   }
}
```

However, all this will be for nothing if we do not allow the peer to actually write and/or read from the characteristic value. This is done by adding two lines to ble_cus_init() in ble_cus.c before the call to sd_ble_gatts_service_add(...)
```C
/* This code belongs in custom_value_char_add() in ble_cus.c */
    ...
    BLE_GAP_CONN_SEC_MODE_SET_OPEN(&attr_md.read_perm);
    BLE_GAP_CONN_SEC_MODE_SET_OPEN(&attr_md.write_perm);

    err_code = sd_ble_gatts_characteristic_add(p_cus->service_handle, &char_md,
                                               &attr_char_value,
                                               &p_cus->custom_value_handles);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    return NRF_SUCCESS;
}
```

These two lines sets the write and read permissions to the characteristic value attribute to open, i.e. the peer is allowed to write/read the value without encrypting the link first. Now, try writing to the characteristic using nRF Connect for Desktop or Android/iOS. Every time a value is written to the characteristic, LED4 on the nRF5x DK should toggle.

Write Button  |  Write value    |
------------ | ------------- | 
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_button.jpg" width="250">  | <img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_write_value.jpg" width="250"> |

**Challenge 1:** p_evt_write also has a data field. Use the data to decide if the LED is to be turned on of off. </br>

### Step 8 - Propagating Custom Service events to the application
Until now we've only handled the events that are propagated by the SoftDevice, but in some cases it makes sense to propagate events to the application.</br>
In order to do this we need to add an event handler to our Custom Service Init structure and Custom Service structure
```C
 /* This code belongs in ble_cus.h*/

/**@brief Custom Service init structure. This contains all options and data needed for
 *        initialization of the service.*/
typedef struct
{
    ble_cus_evt_handler_t         evt_handler;                    /**< Event handler to be called for handling events in the Custom Service. */
    uint8_t                       initial_custom_value;           /**< Initial custom value */
    ble_srv_cccd_security_mode_t  custom_value_char_attr_md;     /**< Initial security level for Custom characteristics attribute */
} ble_cus_init_t;

/**@brief Custom Service structure. This contains various status information for the service. */
struct ble_cus_s
{
    ble_cus_evt_handler_t         evt_handler;                    /**< Event handler to be called for handling events in the Custom Service. */
    uint16_t                      service_handle;                 /**< Handle of Custom Service (as provided by the BLE stack). */
    ble_gatts_char_handles_t      custom_value_handles;           /**< Handles related to the Custom Value characteristic. */
    uint16_t                      conn_handle;                    /**< Handle of the current connection (as provided by the BLE stack, is BLE_CONN_HANDLE_INVALID if not in a connection). */
    uint8_t                       uuid_type; 
};
```
*Note that the project will not compile at this point.*

After adding the event handler to the structures we must make sure that we initialize our service correctly in ble_cus_init()
```C
 /* This code belongs in ble_cus_init() in ble_cus.c*/

// Initialize service structure
p_cus->evt_handler               = p_cus_init->evt_handler;
p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;
```

Next, we need to declare an event type specific to our service
```C
 /* This code belongs in ble_cus.h*/

typedef enum
{
    BLE_CUS_EVT_DISCONNECTED,
    BLE_CUS_EVT_CONNECTED
} ble_cus_evt_type_t;
```
For now we're only going to add the BLE_CUS_EVT_CONNECTED and BLE_CUS_EVT_DISCONNECTED events, but we'll add some additional events later in the tutorial.</br>
After declaring the event type we need to declare an event structure that holds a ble_cus_evt_type_t event, i.e.
```C
 /* This code belongs in ble_cus.h*/

/**@brief Custom Service event. */
typedef struct
{
    ble_cus_evt_type_t evt_type;                                  /**< Type of event. */
} ble_cus_evt_t;
```

Next, we need to declare the Custom Service event handler type
```C
 /* This code belongs in ble_cus.h*/

/**@brief Custom Service event handler type. */
typedef void (*ble_cus_evt_handler_t) (ble_cus_t * p_cus, ble_cus_evt_t * p_evt);
```

*If the project doesn't compile at this point in time, you need to change the order of your structs in ble_cus.h. The order should be:*
1. typedef struct ble_cus_s ble_cus_t         // Forward declaration.
2. #BLE_CUS_DEF(_name)                        // Dependent on ble_cus_t.
2. ble_cus_evt_type_t                         // enum. Not dependent on other variables.
3. ble_cus_evt_t                              // Dependent on ble_cus_evt_type_t. Therefore this needs to come after ble_cus_evt_type_t
4. typedef void (* ble_cus_evt_handler_t)     // Dependent on ble_cus_evt_t (and the forwarded declaration of ble_cus_t)
5. ble_cus_init_t                             // Dependent on ble_cus_evt_handler_t
6. ble_cus_s                                  // the structure that describes ble_cus_t that we started with.

Now, back in main.c we're going to create the event handler function on_cus_evt, which takes the same parameters as the ble_cus_evt_handler_t type.

```C
 /* This code belongs in main.c*/

/**@brief Function for handling the Custom Service Service events.
 *
 * @details This function will be called for all Custom Service events which are passed to
 *          the application.
 *
 * @param[in]   p_cus_service  Custom Service structure.
 * @param[in]   p_evt          Event received from the Custom Service.
 *
 */
static void on_cus_evt(ble_cus_t     * p_cus_service,
                       ble_cus_evt_t * p_evt)
{
    switch(p_evt->evt_type)
    {
        case BLE_CUS_EVT_CONNECTED:
            break;

        case BLE_CUS_EVT_DISCONNECTED:
              break;

        default:
              // No implementation needed.
              break;
    }
}
```

Now, in order to propagate events from our service we need to assign the on_cus_evt function as the event handler function of our service when we initialize the Custom Service. This is done by setting the .evt_handler field of the cus_init struct equal to on_cus_evt in services init() in main.c, i.e.

```C
 /* This code belongs in services_init in main.c*/

    // Set the cus event handler
    cus_init.evt_handler                = on_cus_evt;
```
*note*: Make sure that cus_init.evt_handler is not 0 (NULL) when you call `ble_cus_init(&m_cus, &cus_init);` </br>

We can now invoke this event handler from ble_cus.c by calling p_cus->evt_handler(p_cus, &evt) and as the example we'll invoke the event handler when we get the BLE_GAP_EVT_CONNECTED event, i.e. in the on_connect() function. It's fairly straight forward, we simply declare a ble_cus_evt_t variable and set its .evt_type field to the BLE_CUS_EVT_CONNECTED and then invoke the event handler with said event.
```C
 /* This code belongs in ble_cus.c*/

static void on_connect(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    p_cus->conn_handle = p_ble_evt->evt.gap_evt.conn_handle;

    ble_cus_evt_t evt;

    evt.evt_type = BLE_CUS_EVT_CONNECTED;

    p_cus->evt_handler(p_cus, &evt);
}
```
Now you can use this method to receive the Custom Service events in main.c as well as in ble_cus.c.

### Step 9 - Notifying the Custom Value Characteristic
Now we are able to send BLE messages from the phone to the nRF5x DK, but next, we want to be able to send messages from the nRF5x DK to the phone.
We're going to add the ble_cus_custom_value_update function declaration to the ble_cus.h file, which we're going to use to update our Custom Value Characteristic. 

```C
 /* This code belongs in ble_cus.h*/

/**@brief Function for updating the custom value.
 *
 * @details The application calls this function when the cutom value should be updated. If
 *          notification has been enabled, the custom value characteristic is sent to the client.
 *
 * @note 
 *       
 * @param[in]   p_cus          Custom Service structure.
 * @param[in]   Custom value 
 *
 * @return      NRF_SUCCESS on success, otherwise an error code.
 */

ret_code_t ble_cus_custom_value_update(ble_cus_t * p_cus, uint8_t custom_value);
```

Back in ble_cus.c, we're going to continue the good practice of checking that the pointer we passed as an argument isn't NULL
```C
/* This code belongs in ble_cus.c */

ret_code_t ble_cus_custom_value_update(ble_cus_t * p_cus, uint8_t custom_value)
{
    if (p_cus == NULL)
    {
        return NRF_ERROR_NULL;
    }
}
```

Next we're going to update the value in the GATT table with our custom_value that we passed as an argument to the ble_cus_custom_value_update() function

```C
/* This code belongs in ble_cus_custom_value_update() in ble_cus.c*/

    ret_code_t err_code = NRF_SUCCESS;
    ble_gatts_value_t gatts_value;

    // Initialize value struct.
    memset(&gatts_value, 0, sizeof(gatts_value));

    gatts_value.len     = sizeof(uint8_t);
    gatts_value.offset  = 0;
    gatts_value.p_value = &custom_value;

    // Update database.
    err_code = sd_ble_gatts_value_set(p_cus->conn_handle,
                                        p_cus->custom_value_handles.value_handle,
                                        &gatts_value);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }
```

After updating the value in the GATT table we're ready to notify our peer that the value of our Custom Value Characteristic has changed.
```C
/* This code belongs in ble_cus_custom_value_update() in ble_cus.c*/

        // Send value if connected
    if ((p_cus->conn_handle != BLE_CONN_HANDLE_INVALID)) 
    {
        ble_gatts_hvx_params_t hvx_params;

        memset(&hvx_params, 0, sizeof(hvx_params));

        hvx_params.handle = p_cus->custom_value_handles.value_handle;
        hvx_params.type   = BLE_GATT_HVX_NOTIFICATION;
        hvx_params.offset = gatts_value.offset;
        hvx_params.p_len  = &gatts_value.len;
        hvx_params.p_data = gatts_value.p_value;

        err_code = sd_ble_gatts_hvx(p_cus->conn_handle, &hvx_params);
    }
    else
    {
        err_code = NRF_ERROR_INVALID_STATE;
    }

    return err_code;

```

The first thing we should do is to verify that we have a valid connection handle, i.e. that we're actually connected to a peer. If not, we should return an error indicating that we're in an invalid state. Nect, we set the ble_gatts_hvx_params_t, which contains the handle of the custom value attribute, the type (i.e. Notification or Indication), the value offset (only used if it's larger than the MTU for the connection, but don't worry about that just yet), and the length of the data.After setting the hvx_params, we notify the peer by calling sd_ble_gatts_hvx(). Lastly, we should return the error_code.</br>

After adding the code snippets above ble_cus_custom_value_update should look like below
```C
ret_code_t ble_cus_custom_value_update(ble_cus_t * p_cus, uint8_t custom_value)
{
    if (p_cus == NULL)
    {
        return NRF_ERROR_NULL;
    }

    ret_code_t err_code = NRF_SUCCESS;
    ble_gatts_value_t gatts_value;

    // Initialize value struct.
    memset(&gatts_value, 0, sizeof(gatts_value));

    gatts_value.len     = sizeof(uint8_t);
    gatts_value.offset  = 0;
    gatts_value.p_value = &custom_value;

    // Update database.
    err_code = sd_ble_gatts_value_set(p_cus->conn_handle,
                                        p_cus->custom_value_handles.value_handle,
                                        &gatts_value);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }
    
    // Send value if connected
    if ((p_cus->conn_handle != BLE_CONN_HANDLE_INVALID)) 
    {
        ble_gatts_hvx_params_t hvx_params;

        memset(&hvx_params, 0, sizeof(hvx_params));

        hvx_params.handle = p_cus->custom_value_handles.value_handle;
        hvx_params.type   = BLE_GATT_HVX_NOTIFICATION;
        hvx_params.offset = gatts_value.offset;
        hvx_params.p_len  = &gatts_value.len;
        hvx_params.p_data = gatts_value.p_value;

        err_code = sd_ble_gatts_hvx(p_cus->conn_handle, &hvx_params);
    }
    else
    {
        err_code = NRF_ERROR_INVALID_STATE;
    }

    return err_code;
}
```
The CCCD *(Client Characteristic Configuration Descriptor)* is a field which is used to enable notifications *(CCCD = 0x01)* or Indications *(CCCD = 0x02)*.
Like the characteristic metadata we need to set the metadata of the CCCD in custom_value_char_add(), which is done by adding the following snippet before the characteristic metadata is set.

```C
/* This code belongs in custom_value_char_add() in ble_cus.c*/

    memset(&cccd_md, 0, sizeof(cccd_md));

    //  Read  operation on Cccd should be possible without authentication.
    BLE_GAP_CONN_SEC_MODE_SET_OPEN(&cccd_md.read_perm);
    BLE_GAP_CONN_SEC_MODE_SET_OPEN(&cccd_md.write_perm);
    
    cccd_md.vloc       = BLE_GATTS_VLOC_STACK;
```
We're setting the read and write permissions to the CCCD to open, i.e. no encryption is needed to write or read from the CCCD. The .vloc option is set to BLE_GATTS_VLOCK_STACK, which means that the value of the CCCD will be stored in the SoftDevice memory section, and not in the application memory section.</br>
The next step is to add the Notify property to the Custom Value Characteristic and add a pointer to the CCCD metadata in the characteristic metadata. This is done by **modifying** the characteristic metadata properties in the custom_value_char_add() function, i.e. setting .notify to 1 and .p_cccd_md to point to the CCCD metadata struct.
```C
/* This code belongs in custom_value_char_add() in ble_cus.c*/

    char_md.char_props.notify = 1;  
    char_md.p_cccd_md         = &cccd_md; 
```

This will add a Client Characteristic Configuration Descriptor, or CCCD to the Custom Value Characteristic, which allows us to enable or disable notifications by writing to the CCCD. Notification is by default disabled and in order to enable it we have to write 0x0001 to the CCCD. Remember that every time we write to a characteristic or one of it's descriptors, we will get a Write event, thus we need to handle the case where a peer writes to the CCCD in the on_write() function.</br>

However, before modifying the on_write() function we need to add two additional events, BLE_CUS_EVT_NOTIFICATION_ENABLED and BLE_CUS_EVT_NOTIFICATION_DISABLED, to the ble_cus_evt_type_t enumeration in ble_cus.h
```C
/* This code belongs in ble_cus.h */

/**@brief Custom Service event type. */
typedef enum
{
    BLE_CUS_EVT_NOTIFICATION_ENABLED,                             /**< Custom value notification enabled event. */
    BLE_CUS_EVT_NOTIFICATION_DISABLED,                            /**< Custom value notification disabled event. */
    BLE_CUS_EVT_DISCONNECTED,
    BLE_CUS_EVT_CONNECTED
} ble_cus_evt_type_t;
```

Back to the on_write() function. The first thing we should do is to add an if-statement that checks if the handle that is written to matches the handle of the CCCD and that the value has the correct length. If we pass this check we need to check whether notifications has been enabled or not. 
```C
/* This code belongs in on_write() in ble_cus.c*/

    // Check if the Custom value CCCD is written to and that the value is the appropriate length, i.e 2 bytes.
    if ((p_evt_write->handle == p_cus->custom_value_handles.cccd_handle)
        && (p_evt_write->len == 2)
       )
    {

        // CCCD written, call application event handler
        if (p_cus->evt_handler != NULL)
        {
            ble_cus_evt_t evt;

            if (ble_srv_is_notification_enabled(p_evt_write->data))
            {
                evt.evt_type = BLE_CUS_EVT_NOTIFICATION_ENABLED;
            }
            else
            {
                evt.evt_type = BLE_CUS_EVT_NOTIFICATION_DISABLED;
            }
            // Call the application event handler.
            p_cus->evt_handler(p_cus, &evt);
        }

    }
```

We do not actually need to enable the notification, as this is done automatically by the SoftDevice. However, the SoftDevice will propagate the write event and based on this event we can decide if its ok to start notifying characteristic values or not.</br>
Adding the code-snippet above to the on_write() function should result in the following function
```C
static void on_write(ble_cus_t * p_cus, ble_evt_t const * p_ble_evt)
{
    ble_gatts_evt_write_t * p_evt_write = &p_ble_evt->evt.gatts_evt.params.write;
    
    // Check if the handle passed with the event matches the Custom Value Characteristic handle.
    if (p_evt_write->handle == p_cus->custom_value_handles.value_handle)
    {
        // Put specific task here. 
        if (p_evt_write->data[0] == 0x01)
        {
            nrf_gpio_pin_clear(LED_4);
        }
        else if (p_evt_write->data[0] == 0x00)
        {
            nrf_gpio_pin_set(LED_4);
        }
    }

    // Check if the Custom value CCCD is written to and that the value is the appropriate length, i.e 2 bytes.
    if ((p_evt_write->handle == p_cus->custom_value_handles.cccd_handle)
        && (p_evt_write->len == 2)
       )
    {

        // CCCD written, call application event handler
        if (p_cus->evt_handler != NULL)
        {
            ble_cus_evt_t evt;

            if (ble_srv_is_notification_enabled(p_evt_write->data))
            {
                evt.evt_type = BLE_CUS_EVT_NOTIFICATION_ENABLED;
            }
            else
            {
                evt.evt_type = BLE_CUS_EVT_NOTIFICATION_DISABLED;
            }
            // Call the application event handler.
            p_cus->evt_handler(p_cus, &evt);
        }

    }
}
```
Now, the last thing we have to do is to add the BLE_CUS_EVT_NOTIFICATION_ENABLED and BLE_CUS_EVT_NOTIFICATION_DISABLED to the on_cus_evt() event handler in main.c, i.e.
```C
static void on_cus_evt(ble_cus_t     * p_cus_service,
                       ble_cus_evt_t * p_evt)
{
    switch(p_evt->evt_type)
    {
        case BLE_CUS_EVT_NOTIFICATION_ENABLED:
            break;
        
        case BLE_CUS_EVT_NOTIFICATION_DISABLED:
            break;

        case BLE_CUS_EVT_CONNECTED:
            break;

        case BLE_CUS_EVT_DISCONNECTED:
            break;

        default:
            // No implementation needed.
            break;
    }
}
```

Compile the project and flash it to your nRF5x DK. If you open the nRF Connect app and connect to the device you should see that a Client Characteristic Configuration Descriptor has been added under the characteristic, by the field Properties: NOTIFY, and the three arrows pointing down, as shown in the screen shot below.

Memory Settings Keil | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_notifications.jpg" width="250"> |

We're now ready to notify some values from the nRF5x DK to the nRF Connect app. In order to do that we're going to create an application timer that calls ble_cus_custom_value_update() at a regular interval and then start it when we get the BLE_CUS_EVT_NOTIFICATION_ENABLED event. So first, add the following define to the top of main.c
```C
/* This code belongs in main.c */
#define NOTIFICATION_INTERVAL           APP_TIMER_TICKS(1000)
```

which is going to be the timeout interval of our application timer. Note that APP_TIMER_TICKS() takes the input variable in units given in ms. APP_TIMER_TICKS(1000) gives 1 second. Next you will need to create an application timer id with the name m_notification_timer_id by calling the APP_TIMER_DEF below the defines in main.c, i.e.

```C
/* This code belongs in main.c*/
APP_TIMER_DEF(m_notification_timer_id);
```
Below this macro call you can declare an unsigned 8-bit variable called m_custom_value, i.e.
```C
/* This code belongs in main.c*/
static uint8_t m_custom_value = 0;
```

which will be used to hold the custom value we're going to notify to the nRF Connect app. Next, find the timers_init() function in main.c and add the following snippet to create the application timer
```C
/* This code belongs in timers_init() in main.c*/
    // Create timers.
    err_code = app_timer_create(&m_notification_timer_id, APP_TIMER_MODE_REPEATED, notification_timeout_handler);
    APP_ERROR_CHECK(err_code);
```
Great. We now have an application timer and the next step is to start the application timer when we receive the BLE_CUS_EVT_NOTIFICATION_ENABLED event, i.e. add the following snippet under the BLE_CUS_EVT_NOTIFICATION_ENABLED case in the on_cus_evt event handler in main.c. Remember to declare the variable `ret_code_t err_code;` before `switch(p_evt->evt_type)`
```C
/* This code belongs in on_cus_evt() in main.c*/         
           err_code = app_timer_start(m_notification_timer_id, NOTIFICATION_INTERVAL, NULL);
           APP_ERROR_CHECK(err_code);
```
Similarly, we want the notification timer to stop when we get the BLE_CUS_EVT_NOTIFICATION_DISABLED event. Thus, we add the following snippet to the BLE_GUS_EVT_NOTIFICATION_DISABLED case in the on_cus_evt event handler in main.c
```C
/* This code belongs in on_cus_evt() in main.c*/           
           err_code = app_timer_stop(m_notification_timer_id);
           APP_ERROR_CHECK(err_code);
```
Lastly, we need to create the notification_timeout_handler which will increment the m_custom_value variable and then call ble_cus_custom_value_update by declaring the following function above timers_init() in main.c
```C
/* This code belongs in main.c*/  

/**@brief Function for handling the Battery measurement timer timeout.
 *
 * @details This function will be called each time the battery level measurement timer expires.
 *
 * @param[in] p_context  Pointer used for passing some arbitrary information (context) from the
 *                       app_start_timer() call to the timeout handler.
 */
static void notification_timeout_handler(void * p_context)
{
    UNUSED_PARAMETER(p_context);
    ret_code_t err_code;
    
    // Increment the value of m_custom_value before nortifing it.
    m_custom_value++;
    
    err_code = ble_cus_custom_value_update(&m_cus, m_custom_value);
    APP_ERROR_CHECK(err_code);
}
```
Compile the project and flash it to your nRF5x DK. Open the nRF Connect app, connect to the nRF5x DK and enable notification by clicking the button highlighted in the screenshot below. It should change from grey to blue.

Memory Settings Keil | 
------------ |
<img src="https://github.com/edvinand/custom_ble_service_example/blob/master/images/nRF_Connect_notifications.jpg" width="250"> |

You should now see a value field appear below the "Properties" field and the value should be incrementing every second. Congratulations! You have now created a custom service and notified custom values!
