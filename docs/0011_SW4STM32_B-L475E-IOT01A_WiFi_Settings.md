# SW4STM32 B-L475E-IOT01A WiFi Settings

```C
int main(void)
{ 
    ... 

    /* Working application */
    iothub_mqtt_client_run();

    return 0;
}

int iothub_mqtt_client_run(void)
{
    if (platform_init() != 0)
    {
        (void)printf("platform_init failed\r\n");
        return __FAILURE__;
    }
    else
    {
        ...
    }
}

int platform_init(void)
{
    ...
    if (net_init(&hnet, NET_IF, (net_if_init)) != NET_OK)
    {
        CLOUD_Error_Handler(CLOUD_DEMO_IP_ADDRESS_ERROR);
        return -1;
    }
    ...
}

int net_if_init(void * if_ctxt)
{
    ...
    printf("\n*** WIFI connection ***\n\n");

    while (checkWiFiCredentials(&ssid, &psk, (uint8_t *) &security_mode) != HAL_OK)
    {
        printf("Your WIFI parameters need to be entered to proceed\n");
        updateWiFiCredentials(&ssid, &psk, (uint8_t *) &security_mode);
    }
    ...
}

/**
 * @brief  Write the Wifi parameters to the FLASH memory.
 * @param  In:   ssid            Wifi SSID.
 * @param  In:   psk             Wifi security password.
 * @param  In:   security_mode   See @ref wifi_network_security_t definition.
 * @retval	Error code
 * 			0	Success
 * 			<0	Unrecoverable error
 */
int updateWiFiCredentials(const char ** const ssid, const char ** const psk, uint8_t * const security_mode)
{
    wifi_config_t wifi_config;
    int ret = 0;

    if ((ssid == NULL) ||(psk == NULL) || (security_mode == NULL))
    {
        return -1;
    }

    memset(&wifi_config, 0, sizeof(wifi_config_t));

    msg_info("\nEnter SSID: ");

    getInputString(wifi_config.ssid, USER_CONF_WIFI_SSID_MAX_LENGTH);
    msg_info("You have entered %s as the ssid.\n", wifi_config.ssid);

    msg_info("\n");
    char c;
    do
    {
        msg_info("\rEnter Security Mode (0 - Open, 1 - WEP, 2 - WPA, 3 - WPA2): \b");
        c = getchar();
    }
    while ( (c < '0')  || (c > '3'));
    wifi_config.security_mode = c - '0';
    msg_info("\nYou have entered %d as the security mode.\n", wifi_config.security_mode);


    if (wifi_config.security_mode != 0)
    {
        msg_info("\nEnter password: "); 
        getInputString(wifi_config.psk, sizeof(wifi_config.psk));
    }

    wifi_config.magic = USER_CONF_MAGIC;


    ret = FLASH_update((uint32_t)&lUserConfig.wifi_config, &wifi_config, sizeof(wifi_config_t));  

    if (ret < 0)
    {
        msg_error("Failed updating the wifi configuration in FLASH.\n");
    }

    msg_info("\n");
    return ret;
}

/**
 * @brief  Get a line from the console (user input).
 * @param  Out:  inputString   Pointer to buffer for input line.
 * @param  In:   len           Max length for line.
 * @retval Number of bytes read from the terminal.
 */
int getInputString(char *inputString, size_t len)
{
    size_t currLen = 0;
    int c = 0;

    c = getchar();

    while ((c != EOF) && ((currLen + 1) < len) && (c != '\r') && (c != '\n') )
    {
        if (c == '\b')
        {
            if (currLen != 0)
            {
                --currLen;
                inputString[currLen] = 0;
                msg_info(" \b");
            }
        }
        else
        {
            if (currLen < (len-1))
            {
                inputString[currLen] = c;
            }

            ++currLen;
        }
        c = getchar();
    }
    if (currLen != 0)
    { /* Close the string in the input buffer... only if a string was written to it. */
        inputString[currLen] = '\0';
    }
    if (c == '\r')
    {
        c = getchar(); /* assume there is '\n' after '\r'. Just discard it. */
    }

    return currLen;
}
```
