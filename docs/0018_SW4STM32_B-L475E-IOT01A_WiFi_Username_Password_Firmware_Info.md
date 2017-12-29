# SW4STM32 B-L475E-IOT01A Username Password Firmware Info

## Username Password

```C
#ifdef USE_WIFI
  //  /* Possibility to update the parameters if the user button is pushed*/
  if (network_check_credential()==0)             -------------------------------------------------------------------+
  {                                                                                                                 |
    msg_info("Press the User button (Blue) within the next 5 seconds if you want to update the configuration\r\n"   |
         "(NetWork or Cloud security credentials)\r\n\r\n");                                                        |
                                                                                                                    |
   if ( Button_WaitForPush(5000) )                                                                                  |
   {                                                                                                                |
      if (check("Do you want to configure the Wifi credentials ? (y/n)\r\n"))  network_enter_credential();          |
   }                                                                                          |                     |
    updated_network_credential=1;                                                             |                     |
                                                                                              |                     |
  }                                                                                           |                     |
#endif                                                                                        |                     |
                                                                                              |                     |
int network_check_credential(void)                           <--------------------------------*---------------------+
{                                                                                             |
  const char *ssid;                                                                           |
  const char *psk;                                                                            |
  uint8_t   security_mode;                                                                    +--------------------+
  return checkWiFiCredentials(&ssid, &psk, &security_mode);                                                        |
}                |                                                                                                 |
                 V                                                                                                 |
int checkWiFiCredentials(const char ** const ssid, const char ** const psk, uint8_t * const security_mode)         |
{                                                                                                                  |
  bool is_ssid_present = 0;                                                                                        |
                                                                                                                   |
  if (lUserConfig.wifi_config.magic == USER_CONF_MAGIC)            <---------------------+                         |
  {                                                                                      |                         |
    is_ssid_present = true;                                                              |                         |
    if ((ssid == NULL) ||(psk == NULL) || (security_mode == NULL))                       |                         |
    {                                                                                    |                         |
      return -2;                                                                         |                         |
    }                                                                                    |                         |
    *ssid = lUserConfig.wifi_config.ssid;                                                |                         |
    *psk = lUserConfig.wifi_config.psk;                                                  |                         |
    *security_mode = lUserConfig.wifi_config.security_mode;                              |                         |
  }                                                                                      |                         |
                                                                                         |                         |
  return (is_ssid_present) ? 0 : -1;                                                     |                         |
}                                                                                        |                         |
                                                                                         |                         |
#ifdef __ICCARM__  /* IAR */                                                             |                         |
__no_init const user_config_t lUserConfig @ "UNINIT_FIXED_LOC";                          |                         |
#elif defined ( __CC_ARM   )/* Keil / armcc */                                           |                         |
user_config_t lUserConfig __attribute__((section("UNINIT_FIXED_LOC"), zero_init));       |                         |
#elif defined ( __GNUC__ )      /*GNU Compiler */                                        |                         |
user_config_t lUserConfig __attribute__((section("UNINIT_FIXED_LOC")));            <-----+ refer to ld file ---+-----+
#endif                                                                                                         |   | |
                                                                                                               |   | |
typedef struct {                                                                                               |   | |
  char tls_root_ca_cert[USER_CONF_TLS_OBJECT_MAX_SIZE * 3]; /* Allow room for 3 root CA certificates */        |   | |
  char tls_device_cert[USER_CONF_TLS_OBJECT_MAX_SIZE];                                                         |   | |
  // Above structure member must be aligned on 256 bye  boundary                                               |   | |
   // to match firewall constraint , same for the size.                                                        |   | |
  char tls_device_key[USER_CONF_TLS_OBJECT_MAX_SIZE];                                                          |   | |
  wifi_config_t wifi_config;                                               -----------+                        |   | |
  iot_config_t iot_config;                                                 -----------*---+                    |   | |
  uint64_t tls_magic;                          /**< The USER_CONF_MAGIC magic word signals that the TLS strings above was once written to FLASH. */
} user_config_t;                  <---------------------------------------------------*---*--------------------+   | |
                                                                                      |   |                        | |
typedef struct {                                                                      |   |                        | |
  uint64_t magic;                             /**< The USER_CONF_MAGIC magic word signals that the structure was once written to FLASH. */
  char ssid[USER_CONF_WIFI_SSID_MAX_LENGTH];  /**< Wifi network SSID. */              |   |                        | |
  char psk[USER_CONF_WIFI_PSK_MAX_LENGTH];    /**< Wifi network PSK. */               |   |                        | |
  uint8_t security_mode;                      /**< Wifi network security mode. See @ref wifi_network_security_t definition. */
} wifi_config_t;                         <--------------------------------------------+   |                        | |
                                                                                          |                        | |
typedef struct {                                                                          |                        | |
  uint64_t magic;                             /**< The USER_CONF_MAGIC magic word signals that the structure was once written to FLASH. */
  char device_name[USER_CONF_DEVICE_NAME_LENGTH];                                         |                        | |
  char server_name[USER_CONF_SERVER_NAME_LENGTH];                                         |                        | |
  char user_name[USER_CONF_SERVER_NAME_LENGTH];                                           |                        | |
  char password[USER_CONF_SERVER_NAME_LENGTH];                                            |                        | |
} iot_config_t;                         <-------------------------------------------------+                        | |
                                                                                                                   | |
void network_enter_credential(void)                                                                                | |
{                                                                                                                  | |
  const char *ssid;                                                                                                | |
  const char  *psk;                                                                                                | |
  WIFI_Ecn_t security_mode;                                                                                        | |
                                                                                                                   | |
  updateWiFiCredentials(&ssid, &psk, (uint8_t *) &security_mode);                                                  | |
}                                                                                                                  | |
                                                                                                                   | |
int updateWiFiCredentials(const char ** const ssid, const char ** const psk, uint8_t * const security_mode) <------+ |
{                                                                                                                    |
  wifi_config_t wifi_config;                                                                                         |
  int ret = 0;                                                                                                       |
                                                                                                                     |
  if ((ssid == NULL) ||(psk == NULL) || (security_mode == NULL))                                                     |
  {                                                                                                                  |
    return -1;                                                                                                       |
  }                                                                                                                  |
                                                                                                                     |
  memset(&wifi_config, 0, sizeof(wifi_config_t));                                                                    |
                                                                                                                     |
  msg_info("\r\nEnter SSID: ");                                                                                      |
                                                                                                                     |
  getInputString(wifi_config.ssid, USER_CONF_WIFI_SSID_MAX_LENGTH);                                                  |
  msg_info("You have entered %s as the ssid.\r\n", wifi_config.ssid);                                                |
                                                                                                                     |
  msg_info("\r\n");                                                                                                  |
  char c;                                                                                                            |
  do                                                                                                                 |
  {                                                                                                                  |
      msg_info("\rEnter Security Mode (0 - Open, 1 - WEP, 2 - WPA, 3 - WPA2): \b");                                  |
      c = getchar();                                                                                                 |
  }                                                                                                                  |
  while ( (c < '0')  || (c > '3'));                                                                                  |
  wifi_config.security_mode = c - '0';                                                                               |
  msg_info("\r\nYou have entered %d as the security mode.\r\n", wifi_config.security_mode);                          |
                                                                                                                     |
  if (wifi_config.security_mode != 0)                                                                                |
  {                                                                                                                  |
    msg_info("\r\nEnter password: ");                                                                                |
    getInputString(wifi_config.psk, sizeof(wifi_config.psk));                                                        |
  }                                                                                                                  |
                                                                                                                     |
  wifi_config.magic = USER_CONF_MAGIC;                                                                               |
                                                                                                                     |
                                                                                                                     |
  ret = FLASH_update((uint32_t)&lUserConfig.wifi_config, &wifi_config, sizeof(wifi_config_t));  -------+             |
                                                                                                       |             |
  if (ret < 0)                                                                                         |             |
  {                                                                                                    |             |
    msg_error("Failed updating the wifi configuration in FLASH.\r\n");                                 |             |
  }                                                                                                    |             |
                                                                                                       |             |
  msg_info("\r\n");                                                                                    |             |
  return ret;                                                                                          |             |
}                                                                                                      |             |
                                                                                                       |             |
int FLASH_update(uint32_t dst_addr, const void *data, uint32_t size)           <-----------------------+             |
{                                                                                                                    |
  int ret = 0;                                                                                                       |
  int remaining = size;                                                                                              |
  uint8_t * src_addr = (uint8_t *) data;                                                                             |
  uint64_t page_cache[FLASH_PAGE_SIZE/sizeof(uint64_t)];                                                             |
                                                                                                                     |
    __HAL_FLASH_CLEAR_FLAG(FLASH_FLAG_ALL_ERRORS);                                                                   |
                                                                                                                     |
  do {                                                                                                               |
    uint32_t fl_addr = ROUND_DOWN(dst_addr, FLASH_PAGE_SIZE);                                                        |
    int fl_offset = dst_addr - fl_addr;                                                                              |
    int len = MIN(FLASH_PAGE_SIZE - fl_offset, size);                                                                |
                                                                                                                     |
    /* Load from the flash into the cache */                                                                         |
    memcpy(page_cache, (void *) fl_addr, FLASH_PAGE_SIZE);                                                           |
    /* Update the cache from the source */                                                                           |
    memcpy((uint8_t *)page_cache + fl_offset, src_addr, len);                                                        |
    /* Erase the page, and write the cache */                                                                        |
    ret = FLASH_unlock_erase(fl_addr, FLASH_PAGE_SIZE);                                                              |
    if (ret != 0)                                                                                                    |
    {                                                                                                                |
      printf("Error erasing at 0x%08lx\n", fl_addr);                                                                 |
    }                                                                                                                |
    else                                                                                                             |
    {                                                                                                                |
      ret = FLASH_write_at(fl_addr, page_cache, FLASH_PAGE_SIZE);                                                    |
      if(ret != 0)                                                                                                   |
      {                                                                                                              |
        printf("Error writing %lu bytes at 0x%08lx\n", FLASH_PAGE_SIZE, fl_addr);                                    |
      }                                                                                                              |
      else                                                                                                           |
      {                                                                                                              |
        dst_addr += len;                                                                                             |
        src_addr += len;                                                                                             |
        remaining -= len;                                                                                            |
      }                                                                                                              |
    }                                                                                                                |
  } while ((ret == 0) && (remaining > 0));                                                                           |
                                                                                                                     |
  return ret;                                                                                                        |
}                                                                                                                    |
```

## STM32L475VGTx_FLASH.ld

```
...                                                                                                                  |
/* Specify the memory areas */                                                                                       |
MEMORY                                                                                                               |
{                                                                                                                    |
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 96K                                                                   |
RAM2 (xrw)      : ORIGIN = 0x10000000, LENGTH = 32K                                                                  |
FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 400K        /* Use only the first bank */                             |
FLASH_UC (r)    : ORIGIN = 0x08064000, LENGTH = 14K        /* Fixed-location area */   <------+                      |
}                                                                                             |                      |
                                                                                              |                      |
/* Define output sections */                                                                  |                      |
SECTIONS                                                                                      |                      |
{                                                                                             |                      |
                                                                                              |                      |
  INITED_FIXED_LOC : ALIGN(0x800)                                                             |                      |
  {                                                                                           |                      |
    *(INITED_FIXED_LOC)               ----------------------------+                           |                      |
  } >FLASH_UC                                                     |                           |                      |
                                                                  |                           |                      |
  /* uninit region must be last or else it is erased */           |                           |                      |
  UNINIT_FIXED_LOC (NOLOAD) : ALIGN(0x800)                        |                           |                      |
  {                                                               |                           |                      |
    *(UNINIT_FIXED_LOC)      <------------------------------------*---------------------------*----------------------+
  } >FLASH_UC                -------------------------------------*---------------------------+
                                                                  |
  /* The startup code goes first into FLASH */                    |
  .isr_vector :                                                   |
  {                                                               |
    . = ALIGN(8);                                                 |
    KEEP(*(.isr_vector)) /* Startup code */                       |
    . = ALIGN(8);                                                 |
  } >FLASH                                                        |
                                                                  |
  ...                                                             |
}                                                                 |
```

## Firmware Version Infomation

```C
#ifdef __ICCARM__  /* IAR */                                      |
const firmware_version_t lFwVersion @ "INITED_FIXED_LOC" = { FW_VERSION_NAME, FW_VERSION_MAJOR, FW_VERSION_MINOR, FW_VERSION_PATCH, __DATE__, __TIME__ };
#elif defined ( __CC_ARM   ) /* Keil / armcc */                   V
const firmware_version_t lFwVersion __attribute__((section("INITED_FIXED_LOC"))) = { FW_VERSION_NAME, FW_VERSION_MAJOR, FW_VERSION_MINOR, FW_VERSION_PATCH, __DATE__, __TIME__ };
#elif defined ( __GNUC__ )      /*GNU Compiler */
const firmware_version_t lFwVersion __attribute__((section("INITED_FIXED_LOC"))) = { FW_VERSION_NAME, FW_VERSION_MAJOR, FW_VERSION_MINOR, FW_VERSION_PATCH, __DATE__, __TIME__ };
#endif                                                                                     |
                                                                                           |
#define FW_VERSION_NAME   "X-CUBE-Baidu"               <-----------------------------------+
#define FW_VERSION_MAJOR  1
#define FW_VERSION_MINOR  0
#define FW_VERSION_PATCH  1
```
