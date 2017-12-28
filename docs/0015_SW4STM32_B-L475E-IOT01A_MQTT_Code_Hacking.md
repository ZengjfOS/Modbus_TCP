# SW4STM32 B-L475E-IOT01A MQTT Send Hakcing

```C
int iothub_mqtt_client_run(void)
{
    if (platform_init() != 0)
    {
        (void)printf("platform_init failed\r\n");
        return __FAILURE__;
    }
    else
    {
        /**
         * typedef struct MQTT_CLIENT_OPTIONS_TAG
         * {
         *     char* clientId;
         *     char* willTopic;
         *     char* willMessage;
         *     char* username;
         *     char* password;
         *     uint16_t keepAliveInterval;
         *     bool messageRetain;
         *     bool useCleanSession;
         *     QOS_VALUE qualityOfServiceValue;
         *     bool log_trace;
         * } MQTT_CLIENT_OPTIONS;
         */
        MQTT_CLIENT_OPTIONS options = { 0 };
        options.clientId = "iotclient2017"; // remember to create a new ID for each device, this id must be unique
        options.willMessage = NULL;
        options.willTopic = NULL;
        options.username = USERNAME;
        options.password = PASSWORD;
        options.keepAliveInterval = 10;
        options.useCleanSession = true;
        /**
         * 1. At most once: QoS = 0;
         * 2. At least once: Qos = 1;
         * 3. Exactly once: QoS = 2;
         */
        options.qualityOfServiceValue = DELIVER_AT_MOST_ONCE;           

        // #define ENDPOINT "zengjf.mqtt.iot.gz.baidubce.com"
        const char *endpoint = ENDPOINT;

        /**
         * typedef enum MQTT_CONNECTION_TYPE_TAG
         * {
         *     MQTT_CONNECTION_TCP,
         *     MQTT_CONNECTION_TLS
         * } MQTT_CONNECTION_TYPE;
         */
        MQTT_CONNECTION_TYPE type = MQTT_CONNECTION_TCP;

        IOTHUB_CLIENT_RETRY_POLICY retryPolicy = IOTHUB_CLIENT_RETRY_EXPONENTIAL_BACKOFF;

        size_t retryTimeoutLimitInSeconds = 1000;

        // 初始化MQTT连接参数
        IOTHUB_MQTT_CLIENT_HANDLE clientHandle = initialize_mqtt_client_handle(&options, endpoint, type, on_recv_callback,
                                                                               retryPolicy, retryTimeoutLimitInSeconds);

        if (clientHandle == NULL)
        {
            printf("Error: fail to initialize IOTHUB_MQTT_CLIENT_HANDLE");
            return 0;
        }

        // 连接MQTTT Server
        /**
         * // timeout is used to set timeout window for connection in seconds
         * int iothub_mqtt_doconnect(IOTHUB_MQTT_CLIENT_HANDLE iotHubClient, size_t timeout)
         * {
         *     int result = 0;
         *     tickcounter_ms_t currentTime, lastSendTime;
         *     tickcounter_get_current_ms(iotHubClient->msgTickCounter, &lastSendTime);
         * 
         *     do
         *     {
         *         iothub_mqtt_dowork(iotHubClient);
         *         ThreadAPI_Sleep(10);
         *         tickcounter_get_current_ms(iotHubClient->msgTickCounter, &currentTime);
         *     } while (iotHubClient->mqttClientStatus != MQTT_CLIENT_STATUS_CONNECTED
         *              && iotHubClient->isRecoverableError
         *              && (currentTime - lastSendTime) / 1000 <= timeout);
         * 
         *     if ((currentTime - lastSendTime) / 1000 > timeout)
         *     {
         *         LogError("failed to waitfor connection complete in %d seconds", timeout);
         *         result = __FAILURE__;
         *         return result;
         *     }
         * 
         *     if (iotHubClient->mqttClientStatus == MQTT_CLIENT_STATUS_CONNECTED)
         *     {
         *         result = 0;
         *     }
         *     else
         *     {
         *         result = __FAILURE__;
         *     }
         * 
         *     return result;
         * }
         */
        int result = iothub_mqtt_doconnect(clientHandle, 60);

        if (result == __FAILURE__)
        {
            printf("fail to establish connection with server");
            return __FAILURE__;
        }

        /** 
         * typedef struct SUBSCRIBE_PAYLOAD_TAG
         * {
         *     const char* subscribeTopic;
         *     QOS_VALUE qosReturn;
         * } SUBSCRIBE_PAYLOAD;
         * 
         * 1. At most once: QoS = 0;
         * 2. At least once: Qos = 1;
         * 3. Exactly once: QoS = 2;
         */
        SUBSCRIBE_PAYLOAD subscribe[2];
        subscribe[0].subscribeTopic = TOPIC_NAME_A;
        subscribe[0].qosReturn = DELIVER_AT_MOST_ONCE;
        subscribe[1].subscribeTopic = TOPIC_NAME_B;
        subscribe[1].qosReturn = DELIVER_AT_MOST_ONCE;

        /**
         * int subscribe_mqtt_topics(IOTHUB_MQTT_CLIENT_HANDLE iotHubClient, SUBSCRIBE_PAYLOAD *subPayloads, size_t subSize)
         * {
         * 
         *     int result = 0;
         * 
         *     if (subPayloads == NULL || subSize == 0) {
         *         result = __FAILURE__;
         *         LogError("Failure: maybe subPayload is invalid, or subSize is zero.");
         *     }
         *     else
         *     {
         *         bool isQosCheckSuccess = true;
         *         for (int i = 0; i< subSize; ++i)
         *         {
         *             if(subPayloads[i].qosReturn == DELIVER_EXACTLY_ONCE)
         *             {
         *                 LogError("IoT Hub does not support qos = DELIVER_EXACTLY_ONCE");
         *                 result = __FAILURE__;
         *                 isQosCheckSuccess = false;
         *                 break;
         *             }
         *         }
         *         if (isQosCheckSuccess && mqtt_client_subscribe(iotHubClient->mqttClient, GetNextPacketId(iotHubClient), subPayloads, subSize) != 0)
         *         {
         *             LogError("Failure: mqtt_client_subscribe returned error.");
         *             result = __FAILURE__;
         *         }
         *     }
         * 
         *     return result;
         * }
         * 由上面的代码可知，目前不支持DELIVER_EXACTLY_ONCE功能
         */
        subscribe_mqtt_topics(clientHandle, subscribe, sizeof(subscribe)/sizeof(SUBSCRIBE_PAYLOAD));
        const char* publishData = "publish message to topic.";

        // 不需要ACK
        result = publish_mqtt_message(clientHandle, TOPIC_NAME_B, DELIVER_AT_MOST_ONCE, (const uint8_t*)publishData,
                                      strlen(publishData), NULL , NULL);

        // 需要ACK
        /**
         * int pub_least_ack_process(MQTT_PUB_STATUS_TYPE status, void* context)
         * {
         *     IOTHUB_MQTT_CLIENT_HANDLE clientHandle = (IOTHUB_MQTT_CLIENT_HANDLE)context;
         * 
         *     printf("call publish at least once handle\r\n");
         *     if (clientHandle->mqttClientStatus == MQTT_CLIENT_STATUS_CONNECTED)
         *     {
         *         printf("hub is connected\r\n");
         *     }
         * 
         *     if (status == MQTT_PUB_SUCCESS)
         *     {
         *         printf("received publish ack from mqtt server when deliver at least once message\r\n");
         *     }
         *     else
         *     {
         *         printf("fail to publish message to mqtt server\r\n");
         *     }
         * 
         *     return 0;
         * }
         */
        result = publish_mqtt_message(clientHandle, TOPIC_NAME_A, DELIVER_AT_LEAST_ONCE, (const uint8_t*)publishData,
                                      strlen(publishData), pub_least_ack_process , clientHandle);

        TICK_COUNTER_HANDLE tickCounterHandle = tickcounter_create();
        tickcounter_ms_t currentTime, lastSendTime;
        tickcounter_get_current_ms(tickCounterHandle, &lastSendTime);
        bool needSubscribeTopic = false;

        /**
         * #define CREATE_MODEL_INSTANCE(schemaNamespace, ...) \
         *     IF(DIV2(COUNT_ARG(__VA_ARGS__)), CREATE_DEVICE_WITH_INCLUDE_PROPERTY_PATH, CREATE_DEVICE_WITHOUT_INCLUDE_PROPERTY_PATH) (schemaNamespace, __VA_ARGS__)
         * 
         * 注意下面的C2是后面的宏合成的函数，跟代码的无法直接定位的
         * #define IF(condition, trueBranch, falseBranch) C2(IF,ISZERO(condition))(trueBranch, falseBranch)
         * 
         * BEGIN_NAMESPACE(BaiduIotMqttThing);
         * 
         * DECLARE_MODEL(BaiduSerializableIotMqttDev_t,
         *     WITH_DATA(float, temperature),
         *     WITH_DATA(float, humidity),
         *     WITH_DATA(float, pressure),
         *     WITH_DATA(int, proximity),
         *     WITH_DATA(int, accX),
         *     WITH_DATA(int, accY),
         *     WITH_DATA(int, accZ),
         *     WITH_DATA(float, gyrX),
         *     WITH_DATA(float, gyrY),
         *     WITH_DATA(float, gyrZ),
         *     WITH_DATA(int, magX),
         *     WITH_DATA(int, magY),
         *     WITH_DATA(int, magZ)
         *   );
         * 
         * END_NAMESPACE(BaiduIotMqttThing);
         * 
         * #define DECLARE_MODEL(name, ...)                                                             \
         *     REFLECTED_MODEL(name)                                                                    \
         *     FOR_EACH_1(CREATE_DESIRED_PROPERTY_CALLBACK, __VA_ARGS__)                                \
         *     typedef struct name { int :1; FOR_EACH_1(BUILD_MODEL_STRUCT, __VA_ARGS__) } name;        \
         *     FOR_EACH_1_KEEP_1(CREATE_MODEL_ELEMENT, name, __VA_ARGS__)                               \
         *     TO_AGENT_DATA_TYPE(name, DROP_FIRST_COMMA_FROM_ARGS(EXPAND_MODEL_ARGS(__VA_ARGS__)))     \
         *     int FromAGENT_DATA_TYPE_##name(const AGENT_DATA_TYPE* source, void* destination)         \
         *     {                                                                                        \
         *         (void)source;                                                                        \
         *         (void)destination;                                                                   \
         *         LogError("SHOULD NOT GET CALLED... EVER");                                           \
         *         return 0;                                                                            \
         *     }                                                                                        \
         *     static void C2(GlobalInitialize_, name)(void* destination)                               \
         *     {                                                                                        \
         *         (void)destination;                                                                   \
         *         FOR_EACH_1_KEEP_1(CREATE_MODEL_ELEMENT_GLOBAL_INITIALIZE, name, __VA_ARGS__)         \
         *     }                                                                                        \
         *     static void C2(GlobalDeinitialize_, name)(void* destination)                             \
         *     {                                                                                        \
         *         (void)destination;                                                                   \
         *         FOR_EACH_1_KEEP_1(CREATE_MODEL_ELEMENT_GLOBAL_DEINITIALIZE, name, __VA_ARGS__)       \
         *     }                                                                                        \
         */
        BaiduSerializableIotMqttDev_t* mdl = CREATE_MODEL_INSTANCE(BaiduIotMqttThing, BaiduSerializableIotMqttDev_t, true);
        unsigned char* destination;
        size_t destinationSize;

        do
        {
            iothub_mqtt_dowork(clientHandle);
            // 获取当前时间
            tickcounter_get_current_ms(tickCounterHandle, &currentTime);
            // 断线重新设置订阅标志
            if (clientHandle->isConnectionLost && !needSubscribeTopic)
            {
                needSubscribeTopic = true;
            }

            // 断线重新订阅
            if (clientHandle->mqttClientStatus == MQTT_CLIENT_STATUS_CONNECTED && needSubscribeTopic)
            {
                needSubscribeTopic = false;
                subscribe_mqtt_topics(clientHandle, subscribe, sizeof(subscribe)/sizeof(SUBSCRIBE_PAYLOAD));
            }

            // send a publish message every 5 seconds
            // 在连接正常的基础上，发送数据
            if (!clientHandle->isConnectionLost && (currentTime - lastSendTime) / 1000 > 2)
            {
                int16_t  ACC_Value[3];
                float    GYR_Value[3];
                int16_t  MAG_Value[3];

                mdl->temperature = BSP_TSENSOR_ReadTemp();              // 温度
                mdl->humidity = BSP_HSENSOR_ReadHumidity();             // 湿度
                mdl->pressure = BSP_PSENSOR_ReadPressure();             // 大气压力
                mdl->proximity = VL53L0X_PROXIMITY_GetDistance();       // 激光测距

                BSP_ACCELERO_AccGetXYZ(ACC_Value);                      // 加速度计
                mdl->accX = ACC_Value[0];
                mdl->accY = ACC_Value[1];
                mdl->accZ = ACC_Value[2];

                BSP_GYRO_GetXYZ(GYR_Value);                             // 陀螺仪
                mdl->gyrX = GYR_Value[0];
                mdl->gyrY = GYR_Value[1];
                mdl->gyrZ = GYR_Value[2];

                BSP_MAGNETO_GetXYZ(MAG_Value);                          // 磁力计
                mdl->magX = MAG_Value[0];
                mdl->magY = MAG_Value[1];
                mdl->magZ = MAG_Value[2];


                /**
                 * #define SERIALIZE(destination, destinationSize,...) CodeFirst_SendAsync(destination, destinationSize, COUNT_ARG(__VA_ARGS__) FOR_EACH_1(ADDRESS_MACRO, __VA_ARGS__))
                 * 
                 * 序列化JSON数据
                 */
                if (SERIALIZE(&destination, &destinationSize, *mdl) == CODEFIRST_OK)
                {
                    // 发送数据，并且要求由ACK
                	result = publish_mqtt_message(clientHandle, TOPIC_NAME_A, DELIVER_AT_LEAST_ONCE, (const uint8_t*)destination,
                			destinationSize, pub_least_ack_process , clientHandle);

                	// release memory for serialized data
                	free(destination);
                }

                lastSendTime = currentTime;
            }

            ThreadAPI_Sleep(10);
        } while (!clientHandle->isDestroyCalled && !clientHandle->isDisconnectCalled);

        iothub_mqtt_destroy(clientHandle);
        tickcounter_destroy(tickCounterHandle);
        return 0;
    }

#ifdef _CRT_DBG_MAP_ALLOC
    _CrtDumpMemoryLeaks();
#endif
}
```
