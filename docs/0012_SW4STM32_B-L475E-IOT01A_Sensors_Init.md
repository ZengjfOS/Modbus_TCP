# SW4STM32 B-L475E-IOT01A Sensors Init

```C
int main(void)
{
    ...

    /* Working application */
    iothub_mqtt_client_run();  ---------------+
                                              |
    return 0;                                 |
}                                             |
                                              |
int iothub_mqtt_client_run(void)   <----------+
{
    if (platform_init() != 0)      ---------------------+
    {                                                   |
        (void)printf("platform_init failed\r\n");       |
        return __FAILURE__;                             |
    }                                                   |
    else                                                |
    {                                                   |
        ...                                             |
    }                                                   |
}                                                       |
                                                        |
int platform_init(void)         <-----------------------+
{
    ...
#ifdef SENSOR
    int res = init_sensors();   -------------------------------------+
    if(0 != res)                                                     |
    {                                                                |
        msg_error("init_sensors returned error : %d\n", res);        |
    }                                                                |
#endif /* SENSOR */                                                  |
                                                                     |
    return 0;                                                        |
}                                                                    |
                                                                     |
/**                                                                  |
  * @brief  init_sensors                                             |
  * @param  none                                                     |
  * @retval 0 in case of success                                     |
  *         -1 in case of failure                                    |
  */                                                                 |
int init_sensors(void)           <-----------------------------------+
{
    int ret = 0;

    if (HSENSOR_OK != BSP_HSENSOR_Init())
    {
        msg_error("BSP_HSENSOR_Init() returns %d\n", ret);
        ret = -1;
    }

    if (TSENSOR_OK != BSP_TSENSOR_Init())
    {
        msg_error("BSP_TSENSOR_Init() returns %d\n", ret);
        ret = -1;
    }

    if (PSENSOR_OK != BSP_PSENSOR_Init())
    {
        msg_error("BSP_PSENSOR_Init() returns %d\n", ret);
        ret = -1;
    }

    if (MAGNETO_OK != BSP_MAGNETO_Init())
    {
        msg_error("BSP_MAGNETO_Init() returns %d\n", ret);
        ret = -1;
    }

    if (GYRO_OK != BSP_GYRO_Init())
    {
        msg_error("BSP_GYRO_Init() returns %d\n", ret);
        ret = -1;
    }

    if (ACCELERO_OK != BSP_ACCELERO_Init())
    {
        msg_error("BSP_ACCELERO_Init() returns %d\n", ret);
        ret = -1;
    }

    VL53L0X_PROXIMITY_Init();

    return ret;
}
```
