# iot-edge-sdk-for-iot-parser config

## cat gwconfig.txt

```
{
"endpoint":"tcp://parser_endpoint1513157537065.mqtt.iot.gz.baidubce.com:1883",
"topic":"mb_commandTopic_v21513157537065",
"backControlTopic":"mb_backControlTopic_1513157537065",
"user":"parser_endpoint1513157537065/mb_thing_v21513157537065",
"password":"jktSUcYUFC7efnf6nQYh0a4R5ZHUZuOSi/litLFXhXo="
}
```

## cat policyCache.txt：

```
[
    {
        "gatewayId":"5f69af20-79b3-44bf-84ba-3348a0155bf3",
        "trantable":"15c3f8b0-d634-4da0-bfca-2318cbeacd17",
        "functioncode":1,
        "slaveId":5,
        "pubChannel":{
            "endpoint":"tcp://parser_endpoint1513157537065.mqtt.iot.gz.baidubce.com:1883",
            "topic":"mb_dataTopic_v31513581732818",
            "user":"parser_endpoint1513157537065/mb_thing_v21513157537065",
            "password":"cLTFcng6DHjJf+DePzVAV41qKsLDtCmdz8o9Z4looTE="
        },
        "ip_com_addr":"23.106.155.16:5020",
        "databits":8,
        "stopbits":1,
        "interval":3,
        "mode":0,
        "length":8,
        "start_addr":1,
        "parity":"N",
        "baud":300
    }
]
```

## iot-edge-sdk-for-iot-parser Code Hacking

```
int main(int argc, char* argv[])
{
    init_and_start();   -----------+
                                   |
    wait_user_input();             |
                                   |
    clean_and_exit();              |
                                   |
    return 0;                      |
}                                  |
                                   |
void init_and_start()   <----------+
{
    printf("Baidu IoT Edge SDK v0.2.0\r\n");

    init_static_data();

    //
    // 1 load gateway configuration from local file
    // 2 receive device(slave) polling config from cloud, or local cache
    // 3 start execution, modbus read and PUB

    // 1 load gateway configuration from local file
    if (load_gateway_config(&g_gateway_conf))            ------------------------------------+
    {                                                                                        |
        printf("successfully loaded gateway config from file %s\r\n", CONFIG_FILE);          |
    }                                                                                        |
    else                                                                                     |
    {                                                                                        |
        printf("failed to load gateway configuration from file %s\r\n", CONFIG_FILE);        |
    }                                                                                        |
                                                                                             |
    g_mqttsender = new_mqtt_sender(DATA_CACHE, g_cache_size);          ----------------------*---+
                                                                                             |   |
    // 2 receive device(slave) polling config from cloud, or local cache                     |   |
    g_slave_header.next = NULL;                                                              |   |
    load_slave_policy_from_cache(&g_slave_header);                     ----------------------*---*-+
                                                                                             |   | |
    start_listen_command();                                            ----------------------*---*-*-+
    start_worker();                                                    ----------------------*---*-*-*-----------+
}                                                                                            |   | | |           |
                                                                                             |   | | |           |
GatewayConfig g_gateway_conf;                  <---------------------------------------------+   | | |           |
                                                                                             |   | | |           |
typedef struct                                                                               |   | | |           |
{                                                                                            |   | | |           |
    char endpoint[MAX_LEN];                                                                  |   | | |           |
    char topic[MAX_LEN];                                                                     |   | | |           |
    char user[MAX_LEN];                                                                      |   | | |           |
    char password[MAX_LEN];                                                                  |   | | |           |
    char backControlTopic[MAX_LEN];                                                          |   | | |           |
} GatewayConfig;                               <---------------------------------------------+   | | |           |
                                                                                             |   | | |           |
int load_gateway_config(GatewayConfig* conf)   <---------------------------------------------+   | | |           |
{                                                                                                | | |           |
    if (conf == NULL)                                                                            | | |           |
    {                                                                                            | | |           |
        printf("the parameter conf is NULL\n");                                                  | | |           |
        return 0;                                                                                | | |           |
    }                                                                                            | | |           |
    char* content = NULL;               v------------------------------------------------------+ | | |           |
    long size = read_file_as_string(CONFIG_FILE, &content);                                    | | | |           |
    if (size <= 0)                                                                             | | | |           |
    {                                                                                          | | | |           |
        printf("failed to open config file %s, while trying to load gateway configuration\n",  | | | |           |
             CONFIG_FILE);                                                                     | | | |           |
        return 0;                                                                              | | | |           |
    }                                                                                          | | | |           |
                                                                                               | | | |           |
    cJSON* root = cJSON_Parse(content);                                                        | | | |           |
    if (root == NULL)                                                                          | | | |           |
    {                                                                                          | | | |           |
        printf("the config file is not a valid json object, file=%s\n", CONFIG_FILE);          | | | |           |
        return 0;                                                                              | | | |           |
    }                                                                                          | | | |           |
    mystrncpy(conf->endpoint, cJSON_GetObjectItem(root, "endpoint")->valuestring, MAX_LEN);    | | | |           |
    mystrncpy(conf->topic, cJSON_GetObjectItem(root, "topic")->valuestring, MAX_LEN);          | | | |           |
    mystrncpy(conf->user, cJSON_GetObjectItem(root, "user")->valuestring, MAX_LEN);            | | | |           |
    mystrncpy(conf->password, cJSON_GetObjectItem(root, "password")->valuestring, MAX_LEN);    | | | |           |
    // backControlTopic may be missing, or may be null                                         | | | |           |
    conf->backControlTopic[0] = 0;                                                             | | | |           |
    if (cJSON_HasObjectItem(root, "backControlTopic")) {                                       | | | |           |
        cJSON* backControlTopicObj = cJSON_GetObjectItem(root, "backControlTopic");            | | | |           |
        if (! cJSON_IsNull(backControlTopicObj)) {                                             | | | |           |
            mystrncpy(conf->backControlTopic, backControlTopicObj->valuestring, MAX_LEN);      | | | |           |
        }                                                                                      | | | |           |
    }                                                                                          | | | |           |
                                                                                               | | | |           |
    if (cJSON_HasObjectItem(root, "misc")) {                                                   | | | |           |
        cJSON* misc = cJSON_GetObjectItem(root, "misc");                                       | | | |           |
        if (misc != NULL) {                                                                    | | | |           |
            g_misc = cJSON_Duplicate(misc, 1);                                                 | | | |           |
        }                                                                                      | | | |           |
    }                                                                                          | | | |           |
                                                                                               | | | |           |
    // g_cache_size                                                                            | | | |           |
    if (cJSON_HasObjectItem(root, "cacheSize")) {                                              | | | |           |
        cJSON* cacheSize = cJSON_GetObjectItem(root, "cacheSize");                             | | | |           |
        if (cacheSize != NULL) {                                                               | | | |           |
            g_cache_size = cacheSize->valueint;                                                | | | |           |
        }                                                                                      | | | |           |
    }                                                                                          | | | |           |
                                                                                               | | | |           |
    free(content);                                                                             | | | |           |
    cJSON_Delete(root);                                                                        | | | |           |
    return 1;                                                                                  | | | |           |
}                                                                                              | | | |           |
                                                                                               | | | |           |
const char* const CONFIG_FILE = "gwconfig.txt";               <--------------------------------+ | | |           |
                                                                                                 | | |           |
int new_mqtt_sender(const char* cacheFile, int cacheSize)     <----------------------------------+ | |           |
{                                                                                                  | |           |
    if (lock_initialized == 0)                                                                     | |           |
    {                                                                                              | |           |
        sender_lock = Thread_create_mutex();                                                       | |           |
        int j = 0;                                                                                 | |           |
        for (; j < MAX_SENDER; j++)                                                                | |           |
        {                                                                                          | |           |
            SENDERS[j] = NULL;                                                                     | |           |
        }                                                                                          | |           |
        lock_initialized = 1;                                                                      | |           |
    }                                                                                              | |           |
    RingBuFi* buf = newRingBuFi(cacheFile, (size_t) cacheSize);                                    | |           |
    MqttSender* sender = (MqttSender*) malloc(sizeof(MqttSender));                                 | |           |
    sender->lock = Thread_create_mutex();                                                          | |           |
    sender->ringbuf = buf;                                                                         | |           |
    sender->mqttClients = NULL;                                                                    | |           |
    sender->badBrokers = NULL;                                                                     | |           |
    sender->status = WORKER_NOT_STARTED;                                                           | |           |
    sender->worker = Thread_start(worker_func, (void*) sender);                                    | |           |
    sender->status = WORKER_RUNNING;                                                               | |           |
    sender->incomingQueue.next = NULL;                                                             | |           |
    Thread_lock_mutex(sender_lock);                                                                | |           |
    int i = 0;                                                                                     | |           |
    int ret = -1;                                                                                  | |           |
    for (; i < MAX_SENDER; i++)                                                                    | |           |
    {                                                                                              | |           |
        if (SENDERS[i] == NULL)                                                                    | |           |
        {                                                                                          | |           |
            SENDERS[i] = sender;                                                                   | |           |
            ret = i;                                                                               | |           |
            break;                                                                                 | |           |
        }                                                                                          | |           |
    }                                                                                              | |           |
    Thread_unlock_mutex(sender_lock);                                                              | |           |
    return ret;                                                                                    | |           |
}                                                                                                  | |           |
                                                                                                   | |           |
SlavePolicy g_slave_header;    // the pure header node for slave polices        <------------------+ |           |
                                                                                                   | |           |
typedef struct SlavePolicy_t                                                    <------------------+ |           |
{                                                                                                  | |           |
    char gatewayid[UUID_LEN];         // the cloud logic gateway id, used to distinguish slaves    | |           |
    int slaveid;                                                                                   | |           |
    ModbusMode mode;                // tcp, rtu, ascii                                             | |           |
    char ip_com_addr[ADDR_LEN];                                                                    | |           |
    char functioncode;                                                                             | |           |
    int start_addr;                                                                                | |           |
    int length;                                                                                    | |           |
    int interval;                    // in seconds                                                 | |           |
    char trantable[UUID_LEN];                                                                      | |           |
    Channel pubChannel;                // which channel to upload(pub) data                        | |           |
    time_t nextRun;                    //  time for next execution of this policy                  | |           |
    struct SlavePolicy_t* next;        // the next salve policy that need to execute               | |           |
    int mqttClient;                                                                                | |           |
    int baud;                                                                                      | |           |
    int databits;                                                                                  | |           |
    char parity;                                                                                   | |           |
    int stopbits;                                                                                  | |           |
} SlavePolicy;                                                                                     | |           |
                                                                                                   | |           |
int load_slave_policy_from_cache(SlavePolicy* header)                   <--------------------------+ |           |
{                                                                                                    |           |
    // in case gateway can't retrieve SlavePolicy from cloud immediately,                            |           |
    // we should cache the polices in a local file. Whenever gateway startup,                        |           |
    // it should load polices form this local cache first, and in the mean time                      |           |
    // listen any policy change pushed from cloud.                                                   |           |
    log_debug("enter loadSlavePolicy");                                                              |           |
    int rc = -1;                                                                                     |           |
    // anyway we will clear the flag that need reload policy                                         |           |
    rc = Thread_lock_mutex(g_policy_update_lock);                                                    |           |
                                                                                                     |           |
    g_policy_updated = 0;                                                                            |           |
                                                                                                     |           |
    char* content = NULL;                                                                            |           |
                                                                                                     |           |
    long filesize = read_file_as_string(POLICY_CACHE, &content);              ---------------------+ |           |
    if (filesize <= 0)                                                                             | |           |
    {                                                                                              | |           |
        printf("failed to open policy cache file %s, skipping policy cache loading\n",             | |           |
                 POLICY_CACHE);                                                                    | |           |
        rc = Thread_unlock_mutex(g_policy_update_lock);                                            | |           |
        return 0;                                                                                  | |           |
    }                                                                                              | |           |
    if (content == NULL)                                                                           | |           |
    {                                                                                              | |           |
        return 0;                                                                                  | |           |
    }                                                                                              | |           |
                                                                                                   | |           |
    rc = Thread_unlock_mutex(g_policy_update_lock);                                                | |           |
    cJSON* fileroot = cJSON_Parse(content);                                                        | |           |
    if (fileroot == NULL)                                                                          | |           |
    {                                                                                              | |           |
        printf("invalid config detected from cache file %s, skipping policy cache loading\r\n",    | |           |
                POLICY_CACHE);                                                                     | |           |
        free(content);                                                                             | |           |
        return 0;                                                                                  | |           |
    }                                                                                              | |           |
    int num = cJSON_GetArraySize(fileroot);                                                        | |           |
    if (num < 0)                                                                                   | |           |
    {                                                                                              | |           |
        printf("no slave policy is loaded from cache file %s\r\n", POLICY_CACHE);                  | |           |
        free(content);                                                                             | |           |
        return 1;                                                                                  | |           |
    }                                                                                              | |           |
                                                                                                   | |           |
    rc = Thread_lock_mutex(g_policy_lock);                                                         | |           |
                                                                                                   | |           |
    // clear all the existing data                                                                 | |           |
    cleanup_data();                                                                                | |           |
                                                                                                   | |           |
    int i = 0;                                                                                     | |           |
    for(i = 0; i < num; i++)                                                                       | |           |
    {                                                                                              | |           |
        cJSON* root = cJSON_GetArrayItem(fileroot, i);                                             | |           |
        SlavePolicy* policy = json_to_slave_poilicy(root);   -------------------+                  | |           |
        init_modbus_context(policy);                                            |                  | |           |
                                                                                |                  | |           |
        // add the policy into list                                             |                  | |           |
        policy->next = g_slave_header.next;                                     |                  | |           |
        g_slave_header.next = policy;                                           |                  | |           |
    }                                                                           |                  | |           |
    rc = Thread_unlock_mutex(g_policy_lock);                                    |                  | |           |
                                                                                |                  | |           |
    cJSON_Delete(fileroot);                                                     |                  | |           |
    free(content);                                                              |                  | |           |
}                                                                               |                  | |           |
                                                                                |                  | |           |
const char* const POLICY_CACHE = "policyCache.txt";                <------------*------------------+ |           |
                                                                                |                    |           |
SlavePolicy* json_to_slave_poilicy(cJSON* root)              <------------------+                    |           |
{                                                                                                    |           |
    SlavePolicy* policy = new_slave_policy();                                                        |           |
    mystrncpy(policy->gatewayid, json_string(root, "gatewayid"), UUID_LEN);                          |           |
    policy->slaveid = json_int(root, "slaveid");                                                     |           |
    int int_mode = json_int(root, "mode");                                                           |           |
    policy->mode = (ModbusMode)int_mode;                                                             |           |
    mystrncpy(policy->ip_com_addr, json_string(root, "ip_com_addr"), ADDR_LEN);                      |           |
    policy->functioncode = (char)json_int(root, "functioncode");                                     |           |
    policy->start_addr = json_int(root, "start_addr");                                               |           |
    policy->length = json_int(root, "length");                                                       |           |
    policy->interval = json_int(root, "interval");                                                   |           |
    mystrncpy(policy->trantable, json_string(root, "trantable"), UUID_LEN);                          |           |
                                                                                                     |           |
    cJSON* cjch = cJSON_GetObjectItem(root, "pubChannel");                                           |           |
    mystrncpy(policy->pubChannel.endpoint, json_string(cjch, "endpoint"), MAX_LEN);                  |           |
    mystrncpy(policy->pubChannel.topic, json_string(cjch, "topic"), MAX_LEN);                        |           |
    mystrncpy(policy->pubChannel.user, json_string(cjch, "user"), MAX_LEN);                          |           |
    mystrncpy(policy->pubChannel.password, json_string(cjch, "password"), MAX_LEN);                  |           |
    policy->nextRun = time(NULL) + policy->interval;                                                 |           |
                                                                                                     |           |
    if (policy->mode == RTU)                                                                         |           |
    {                                                                                                |           |
        policy->baud = json_int(root, "baud");                                                       |           |
        policy->databits = json_int(root, "databits");                                               |           |
        policy->parity = json_string(root, "parity")[0];                                             |           |
        policy->stopbits = json_int(root, "stopbits");                                               |           |
    }                                                                                                |           |
    return policy;                                                                                   |           |
}                                                                                                    |           |
                                                                                                     |           |
void start_listen_command()                                        <---------------------------------+           |
{                                                                                                                |
    printf("connecting gateway to cloud...\r\n");                                                                |
    Thread_lock_mutex(g_gateway_mutex);                                                                          |
    // sub to config mqtt topic, to receive slave policy from cloud                                              |
    MQTTClient client;                                                                                           |
    MQTTClient_connectOptions conn_opts = MQTTClient_connectOptions_initializer;                                 |
    int rc = 0;                                                                                                  |
    int ch = 0;                                                                                                  |
                                                                                                                 |
    char clientid[MAX_LEN];                                                                                      |
    snprintf(clientid, MAX_LEN, "modbusGW%lld", (long long)time(NULL));                                          |
    MQTTClient_create(&client, g_gateway_conf.endpoint, clientid,                                                |
        MQTTCLIENT_PERSISTENCE_NONE, NULL);                                                                      |
                                                                                                                 |
    conn_opts.keepAliveInterval = 20;                                                                            |
    conn_opts.cleansession = 1;                                                                                  |
    conn_opts.username = g_gateway_conf.user;                                                                    |
    conn_opts.password = g_gateway_conf.password;                                                                |
    conn_opts.connectTimeout = 5;                                                                                |
    set_ssl_option(&conn_opts, g_gateway_conf.endpoint);                                                         |
                                                                                                                 |
    MQTTClient_setCallbacks(client, client, connection_lost, msg_arrived, delivered);   ------+                  |
                                                                                              |                  |
    if ((rc = MQTTClient_connect(client, &conn_opts)) != MQTTCLIENT_SUCCESS)                  |                  |
    {                                                                                         |                  |
        MQTTClient_destroy(&client);                                                          |                  |
        printf("Failed to connect, return code %d\r\n", rc);                                  |                  |
        Thread_unlock_mutex(g_gateway_mutex);                                                 |                  |
                                                                                              |                  |
        return;                                                                               |                  |
    }                                                                                         |                  |
    if (strlen(g_gateway_conf.backControlTopic) > 0) {                                        |                  |
        char* topics[2];                                                                      |                  |
        topics[0] = g_gateway_conf.topic;                                                     |                  |
        topics[1] = g_gateway_conf.backControlTopic;                                          |                  |
        int qoss[2];                                                                          |                  |
        qoss[0] = 0;                                                                          |                  |
        qoss[1] = 0;                                                                          |                  |
        MQTTClient_subscribeMany(client, 2, topics, qoss);                                    |                  |
    } else {                                                                                  |                  |
        MQTTClient_subscribe(client, g_gateway_conf.topic, 0);                                |                  |
    }                                                                                         |                  |
                                                                                              |                  |
    g_gateway_connected = 1;                                                                  |                  |
    Thread_unlock_mutex(g_gateway_mutex);                                                     |                  |
}                                                                                             |                  |
                                                                                              |                  |
int msg_arrived(void* context, char* topicName, int topicLen, MQTTClient_message* message) <--+                  |
{                                                                                                                |
    // sometime we receive strange message with topic name like "\300\005@\267"                                  |
    // let's filter those message that with topic other than expected                                            |
    if (message == NULL)                                                                                         |
    {                                                                                                            |
        return 1;                                                                                                |
    }                                                                                                            |
    if (strcmp(g_gateway_conf.topic, topicName) == 0) {                                                          |
        return handle_config_msg(context, topicName, topicLen, message);      ----------------------+            |
    } else if (strlen(g_gateway_conf.backControlTopic) > 0                                          |            |
        && strcmp(g_gateway_conf.backControlTopic, topicName) == 0) {                               |            |
        return handle_back_control_msg(context, topicName, topicLen, message);   -------------------*-+          |
    } else {                                                                                        | |          |
        snprintf(g_buff, BUFF_LEN,                                                                  | |          |
                "received unrelevant message in command topic, skipping it. topic=%s, payload=",    | |          |
                topicName);                                                                         | |          |
        log_debug(g_buff);                                                                          | |          |
                                                                                                    | |          |
        MQTTClient_freeMessage(&message);                                                           | |          |
        MQTTClient_free(topicName);                                                                 | |          |
                                                                                                    | |          |
        return 1;                                                                                   | |          |
    }                                                                                               | |          |
}                                                                                                   | |          |
                                                                                                    | |          |
int handle_config_msg(void* context, char* topicName, int topicLen, MQTTClient_message* message) {<-+ |          |
    int i = 0;                                                                                        |          |
    char* payloadptr = NULL;                                                                          |          |
                                                                                                      |          |
    payloadptr = message->payload;                                                                    |          |
    int buflen = message->payloadlen + 1;                                                             |          |
    char* buf = (char*)malloc(buflen);                                                                |          |
    buf[buflen - 1] = 0;                                                                              |          |
    for(i = 0; i<message->payloadlen; i++)                                                            |          |
    {                                                                                                 |          |
        buf[i] = payloadptr[i];                                                                       |          |
    }                                                                                                 |          |
                                                                                                      |          |
    MQTTClient_freeMessage(&message);                                                                 |          |
    MQTTClient_free(topicName);                                                                       |          |
    cJSON* root = cJSON_Parse(buf);                                                                   |          |
    if (root == NULL)                                                                                 |          |
    {                                                                                                 |          |
        free(buf);                                                                                    |          |
        printf("received invalid json config:%s\r\n", buf);                                           |          |
        return 1;                                                                                     |          |
    }                                                                                                 |          |
    cJSON_Delete(root);                                                                               |          |
    Thread_lock_mutex(g_policy_update_lock);                                                          |          |
    FILE* fp = fopen(POLICY_CACHE, "w");                                                              |          |
    if (! fp)                                                                                         |          |
    {                                                                                                 |          |
        free(buf);                                                                                    |          |
        snprintf(g_buff, BUFF_LEN, "failed to open %s for write", POLICY_CACHE);                      |          |
        log_debug(g_buff);                                                                            |          |
        Thread_unlock_mutex(g_policy_update_lock);                                                    |          |
        return 0;                                                                                     |          |
    }                                                                                                 |          |
    fprintf(fp, "%s", buf);                                                                           |          |
    fclose(fp);                                                                                       |          |
    printf("received gateway config from cloud\r\n");                                                 |          |
    free(buf);                                                                                        |          |
                                                                                                      |          |
    g_policy_updated = 1;                                                                             |          |
    Thread_unlock_mutex(g_policy_update_lock);                                                        |          |
    return 1;                                                                                         |          |
}                                                                                                     |          |
                    v---------------------------------------------------------------------------------+          |
int handle_back_control_msg(void* context, char* topicName, int topicLen, MQTTClient_message* message) {         |
    int i = 1;                                                                                                   |
    char* payloadptr = NULL;                                                                                     |
                                                                                                                 |
    payloadptr = message->payload;                                                                               |
    int buflen = message->payloadlen + 1;                                                                        |
    char* buf = (char*)malloc(buflen);                                                                           |
    buf[buflen - 1] = 0;                                                                                         |
    for(i = 0; i<message->payloadlen; i++)                                                                       |
    {                                                                                                            |
        buf[i] = payloadptr[i];                                                                                  |
    }                                                                                                            |
                                                                                                                 |
    MQTTClient_freeMessage(&message);                                                                            |
    MQTTClient_free(topicName);                                                                                  |
    cJSON* root = cJSON_Parse(buf);                                                                              |
    if (root == NULL)                                                                                            |
    {                                                                                                            |
        free(buf);                                                                                               |
        printf("received invalid json config for writing modbus:%s\r\n", buf);                                   |
        return 1;                                                                                                |
    }                                                                                                            |
                                                                                                                 |
    // the control config looks like                                                                             |
    // {                                                                                                         |
    //     "request1": {                                                                                         |
    //         "slaveid": 1,                                                                                     |
    //         "address": 40001,                                                                                 |
    //         "data": "00ff1234"                                                                                |
    //     },                                                                                                    |
    //     "request2": {                                                                                         |
    //         "slaveid": 2,                                                                                     |
    //         "address": 10020,                                                                                 |
    //         "data": "00ff1234"                                                                                |
    //     }                                                                                                     |
    // }                                                                                                         |
    char key[11];                                                                                                |
    // lets limit the max data point to write to 100                                                             |
    for (i = 1; i <= 100; i++) {                                                                                 |
        sprintf(key, "request%d", i);                                                                            |
        if (cJSON_HasObjectItem(root, key)) {                                                                    |
            cJSON* req = cJSON_GetObjectItem(root, key);                                                         |
            int slaveid = json_int(req, "slaveid");                                                              |
            int address = json_int(req, "address");                                                              |
            char* data = json_string(req, "data");                                                               |
            write_modbus(slaveid, address, data);    -----------------+                                          |
        } else {                                                      |                                          |
            break;                                                    |                                          |
        }                                                             |                                          |
    }                                                                 |                                          |
                                                                      |                                          |
    cJSON_Delete(root);                                               |                                          |
    return 1;                                                         |                                          |
}                                                                     |                                          |
                                                                      |                                          |
int write_modbus(int slaveid, int startAddress, char* data)   <-------+                                          |
{                                                                                                                |
    // check if slave is connected or not                                                                        |
    if (slaveid < 1 || slaveid >= MODBUS_DATA_COUNT)                                                             |
    {                                                                                                            |
        return -1;                                                                                               |
    }                                                                                                            |
    modbus_t* ctx = g_modbus_ctxs[slaveid];                                                                      |
    if (ctx == NULL)                                                                                             |
    {                                                                                                            |
        return -1;                                                                                               |
    }                                                                                                            |
                                                                                                                 |
    // check data                                                                                                |
    if (data == NULL || strlen(data) < 2)                                                                        |
    {                                                                                                            |
        return -1;  // expects at least 2 chars, like "01"                                                       |
    }                                                                                                            |
                                                                                                                 |
    int rc = 0;                                                                                                  |
                                                                                                                 |
    // check start address                                                                                       |
    if (startAddress >= 1 && startAddress < 9999)                                                                |
    {                                                                                                            |
        uint8_t data8[MAX_MODBUS_DATA_TO_WRITE];                                                                 |
        int num = char2uint8(data8, data);                                                                       |
        int startOffset = startAddress - 1;                                                                      |
        if (num > 0)                                                                                             |
        {                                                                                                        |
            rc = modbus_write_bits(ctx, startOffset, num, data8);                                                |
            if (rc == -1)                                                                                        |
            {                                                                                                    |
                printf("write bits failed, slaveid=%d, address=%d, data=%s\n", slaveid, startAddress, data);     |
                return -1;                                                                                       |
            }                                                                                                    |
        }                                                                                                        |
        return 0;                                                                                                |
    }                                                                                                            |
    else if (startAddress >= 40001 && startAddress < 49999)                                                      |
    {                                                                                                            |
        uint16_t data16[MAX_MODBUS_DATA_TO_WRITE];                                                               |
        int num = char2uint16(data16, data);                                                                     |
        int startOffset = startAddress - 40001;                                                                  |
        if (num > 0)                                                                                             |
        {                                                                                                        |
            rc = modbus_write_registers(ctx, startOffset, num, data16);                                          |
            if (rc == -1)                                                                                        |
            {                                                                                                    |
                printf("write registers failed, slaveid=%d, address=%d, data=%s\n", slaveid, startAddress, data);|
                return -1;                                                                                       |
            }                                                                                                    |
        }                                                                                                        |
        return 0;                                                                                                |
    }                                                                                                            |
    else {                                                                                                       |
        // invalid start address                                                                                 |
        printf("unsupported address for write:%d\n", startAddress);                                              |
    }                                                                                                            |
                                                                                                                 |
    return -1;                                                                                                   |
}                                                                                                                |
                                                                                                                 |
void start_worker()                                                   <------------------------------------------+
{
    g_worker_thread = Thread_start(worker_func, (void*) NULL);           ------------------+
}                                                                                          |
                                                                                           |
//TODO: fire up a few more workers, and precess in parallel, to speed up.   <--------------+
static thread_return_type worker_func(void* arg)
{
    g_worker_is_running = 1;
    while (g_stop_worker != 1)
    {
        // load slave policy if it's updated
        if (g_policy_updated)
        {
            load_slave_policy_from_cache(&g_slave_header);
        }

        if (g_gateway_connected == 0)
        {
            start_listen_command();
        }

        // iterate from the beginning of the policy list
        // and pick those whose nextRun is due, and execute them,
        // calculate the new next run, and insert into the list
        time_t now = time(NULL);
        // we have something to do, acquire the lock here
        int rc = Thread_lock_mutex(g_policy_lock);
        if (g_slave_header.next != NULL && g_slave_header.next->nextRun <= now)
        {
            while (g_slave_header.next != NULL && g_slave_header.next->nextRun <= now)
            {
                // detach from the list
                SlavePolicy* policy = g_slave_header.next;
                g_slave_header.next = g_slave_header.next->next;
                execute_policy(policy);        -----------+
            }                                             |
        }                                                 |
        rc = Thread_unlock_mutex(g_policy_lock);          |
                                                          |
        sleep(1);                                         |
    }                                                     |
    log_debug("exiting worker thread...\r\n");            |
    g_worker_is_running = 0;                              |
}                                                         |
                                                          |
                                                          |
void execute_policy(SlavePolicy* policy)       <----------+
{
    // 3 recaculate the next run time
    policy->nextRun = policy->interval + time(NULL);

    // 4 re-insert into the list, in order of nextRun
    insert_slave_policy(policy);

    if (policy == NULL)
    {
        return;
    }

    // 1 query modbus data
    char payload[1024];
    read_modbus(policy, payload);              -----------------+
    // 2 pub modbus data                                        |
    if (strlen(payload) > 0)                                    |
    {                                                           |
        char msgcontent[BUFF_LEN];                              |
        pack_pub_msg(policy, payload, msgcontent);        ------*-----------------------------+
        mqtt_send(g_mqttsender,                           ------*-----------------------------*-+
                    policy->pubChannel.endpoint,                |                             | |
                    policy->pubChannel.user,                    |                             | |
                    policy->pubChannel.password,                |                             | |
                    policy->pubChannel.topic,                   |                             | |
                    msgcontent,                                 |                             | |
                    strlen(msgcontent),                         |                             | |
                    0,                                          |                             | |
                    PEM_FILE);                                  |                             | |
    }                                                           |                             | |
}                                                               |                             | |
                                                                |                             | |
int read_modbus(SlavePolicy* policy, char* payload)  <----------+                             | |
{                                                                                             | |
    if (policy == NULL)                                                                       | |
    {                                                                                         | |
        fprintf(stderr, "NULL policy in read_modbus\n");                                      | |
        return -1;                                                                            | |
    }                                                                                         | |
                                                                                              | |
    // 1 query modbus data                                                                    | |
    // parse the host and port first                                                          | |
    modbus_t* ctx = g_modbus_ctxs[policy->slaveid];                                           | |
    if (ctx == NULL)                                                                          | |
    {                                                                                         | |
        // slave could be offline when we initialize the modbus context,                      | |
        // we need to recover this, by trying to re-connect                                   | |
        fprintf(stderr,                                                                       | |
            "modbus context is NULL in execution phase, trying to reconnect slaveid=%d\n",    | |
            policy->slaveid);                                                                 | |
        init_modbus_context(policy);                                                          | |
                                                                                              | |
        // Ask: slave may disconnect, leaving a non-NULL ctx, how should we recover?          | |
        // Answer: reconnect on communication error                                           | |
    }                                                                                         | |
    ctx = g_modbus_ctxs[policy->slaveid];                                                     | |
    if (ctx == NULL)                                                                          | |
    {                                                                                         | |
        fprintf(stderr, "can't make connection to modbus slave#%d\n", policy->slaveid);       | |
        return -1;                                                                            | |
    }                                                                                         | |
                                                                                              | |
    int start_addr = policy->start_addr;                                                      | |
    int nb = policy->length;                                                                  | |
                                                                                              | |
    uint16_t* tab_rq_registers = NULL;                                                        | |
    uint8_t* tab_rq_bits = NULL;                                                              | |
                                                                                              | |
    int rc = -1;                                                                              | |
    payload[0] = 0;    // empty the payload first                                             | |
    int need_reconnect_modbus = 0;                                                            | |
    switch(policy->functioncode)                                                              | |
    {                                                                                         | |
        case MODBUS_FC_READ_COILS:                                                            | |
            // just store every bit as a byte, for easy of use                                | |
            tab_rq_bits = (uint8_t*) malloc(nb * sizeof(uint8_t));                            | |
            memset(tab_rq_bits, 0, nb * sizeof(uint8_t));                                     | |
            rc = modbus_read_bits(ctx, start_addr, nb, tab_rq_bits);                          | |
            if (rc != nb)                                                                     | |
            {                                                                                 | |
                printf("ERROR modbus_read_bits (%d) slaveid=%d, will reconnect\n",            | |
                     rc, policy->slaveid);                                                    | |
                need_reconnect_modbus = 1;                                                    | |
                break;                                                                        | |
            }                                                                                 | |
            byte_arr_to_hex(payload, (char*)tab_rq_bits, nb * sizeof(uint8_t));   ------------*-*-+
            break;                                                                            | | |
                                                                                              | | |
        case MODBUS_FC_READ_DISCRETE_INPUTS:                                                  | | |
            tab_rq_bits = (uint8_t*) malloc(nb * sizeof(uint8_t));                            | | |
            memset(tab_rq_bits, 0, nb * sizeof(uint8_t));                                     | | |
            rc = modbus_read_input_bits(ctx, start_addr, nb, tab_rq_bits);                    | | |
            if (rc != nb)                                                                     | | |
            {                                                                                 | | |
                printf("ERROR modbus_read_input_bits (%d) slaveid=%d, will reconnect\n",      | | |
                     rc, policy->slaveid);                                                    | | |
                need_reconnect_modbus = 1;                                                    | | |
                break;                                                                        | | |
            }                                                                                 | | |
            byte_arr_to_hex(payload, (char*)tab_rq_bits, nb * sizeof(uint8_t));               | | |
            break;                                                                            | | |
                                                                                              | | |
        case MODBUS_FC_READ_HOLDING_REGISTERS:                                                | | |
            tab_rq_registers = (uint16_t*) malloc(nb * sizeof(uint16_t));                     | | |
            memset(tab_rq_registers, 0, nb * sizeof(uint16_t));                               | | |
            rc = modbus_read_registers(ctx, start_addr, nb, tab_rq_registers);                | | |
            if (rc != nb)                                                                     | | |
            {                                                                                 | | |
                printf("ERROR modbus_read_registers (%d) slaveid=%d, will reconnect\n",       | | |
                     rc, policy->slaveid);                                                    | | |
                need_reconnect_modbus = 1;                                                    | | |
                break;                                                                        | | |
            }                                                                                 | | |
            short_arr_to_array(payload, tab_rq_registers, nb);                                | | |
            break;                                                                            | | |
                                                                                              | | |
        case MODBUS_FC_READ_INPUT_REGISTERS:                                                  | | |
            tab_rq_registers = (uint16_t*) malloc(nb * sizeof(uint16_t));                     | | |
            memset(tab_rq_registers, 0, nb * sizeof(uint16_t));                               | | |
            rc = modbus_read_input_registers(ctx, start_addr, nb, tab_rq_registers);          | | |
            if (rc != nb)                                                                     | | |
            {                                                                                 | | |
                printf("ERROR modbus_read_input_registers (%d) slaveid=%d, will reconnect\n", | | |
                     rc, policy->slaveid);                                                    | | |
                need_reconnect_modbus = 1;                                                    | | |
                break;                                                                        | | |
            }                                                                                 | | |
            short_arr_to_array(payload, tab_rq_registers, nb);                                | | |
            break;                                                                            | | |
                                                                                              | | |
        default:                                                                              | | |
            fprintf(stderr, "not supported function code:%d\n", policy->functioncode);        | | |
            break;                                                                            | | |
    }                                                                                         | | |
    if (tab_rq_bits != NULL)                                                                  | | |
    {                                                                                         | | |
        free(tab_rq_bits);                                                                    | | |
    }                                                                                         | | |
                                                                                              | | |
    if (tab_rq_registers != NULL)                                                             | | |
    {                                                                                         | | |
        free(tab_rq_registers);                                                               | | |
    }                                                                                         | | |
                                                                                              | | |
    if (need_reconnect_modbus == 1)                                                           | | |
    {                                                                                         | | |
        if (g_modbus_ctxs[policy->slaveid] != NULL)                                           | | |
        {                                                                                     | | |
            // in case there is error when read modbus, we need                               | | |
            // to close the current context, otherwise, the                                   | | |
            // port/serial port is still be occupied.                                         | | |
            modbus_close(g_modbus_ctxs[policy->slaveid]);                                     | | |
            modbus_free(g_modbus_ctxs[policy->slaveid]);                                      | | |
        }                                                                                     | | |
        g_modbus_ctxs[policy->slaveid] = NULL;                                                | | |
        init_modbus_context(policy);                                                          | | |
    }                                                                                         | | |
}                                                                                             | | |
                                                                                              | | |
// convert char* to hex, like 0A126F...                                                       | | |
void byte_arr_to_hex(char* dest, char* src, int len)               <--------------------------*-*-+
{                                                                                             | |
    int i = 0;                                                                                | |
    for (; i < len; i++)                                                                      | |
    {                                                                                         | |
        char2hex(src[i], &dest[i << 1], &dest[(i << 1) + 1]);                                 | |
    }                                                                                         | |
    dest[len << 1] = '\0';                                                                    | |
}                                                                                             | |
                                                                                              | |
void pack_pub_msg(SlavePolicy* policy, char* raw, char* dest)          <----------------------+ |
{                                                                                               |
    cJSON* root = cJSON_CreateObject();                                                         |
    cJSON_AddNumberToObject(root, "bdModbusVer", 1);                                            |
    cJSON_AddStringToObject(root, "gatewayid", policy->gatewayid);                              |
    cJSON_AddStringToObject(root, "trantable", policy->trantable);                              |
    cJSON* modbus = NULL;                                                                       |
    cJSON_AddItemToObject(root, "modbus", modbus = cJSON_CreateObject());                       |
                                                                                                |
    cJSON* request = NULL;                                                                      |
    cJSON_AddItemToObject(modbus, "request", request = cJSON_CreateObject());                   |
    cJSON_AddNumberToObject(request, "functioncode", policy->functioncode);                     |
    cJSON_AddNumberToObject(request, "slaveid", policy->slaveid);                               |
    cJSON_AddNumberToObject(request, "startAddr", policy->start_addr);                          |
    cJSON_AddNumberToObject(request, "length", policy->length);                                 |
                                                                                                |
    cJSON_AddStringToObject(modbus, "response", raw);                                           |
                                                                                                |
    time_t now = time(NULL);                                                                    |
    char timestamp[40];                                                                         |
    snprintf(timestamp, 39, "%lld", (long long) now);                                           |
    cJSON_AddStringToObject(root, "timestamp", timestamp);                                      |
                                                                                                |
    // misc                                                                                     |
    if (g_misc != NULL) {                                                                       |
        cJSON* misc = cJSON_Duplicate(g_misc, 1);                                               |
        cJSON_AddItemToObject(root, "misc", misc);                                              |
    }                                                                                           |
    char* text = cJSON_PrintUnformatted(root);                                                  |
    mystrncpy(dest, text, BUFF_LEN);                                                            |
    free(text);                                                                                 |
    cJSON_Delete(root);                                                                         |
}                                                                                               |
                                                                                                |
char mqtt_send(int handle,                  <---------------------------------------------------+
    const char* endpoint,
    const char* username,
    const char* password,
    const char* topic,
    const char* payload,
    int payloadlen,
    char retain,
    const char* certfile)
{
    // 0, check if handle is valid or not
    if (handle < 0 || handle >= MAX_SENDER || SENDERS[handle] == NULL)
    {
        return -1;
    }

    // 1, check if it's a bad broker
    if (isKnownBadBroker(SENDERS[handle], endpoint, username, password) == 1) {
        return -1;
    }

    // add to ring buffer
    MqttMessageToPub* msg = (MqttMessageToPub*) malloc(sizeof(MqttMessageToPub));
    msg->next = NULL;
    int tempLen = strlen(endpoint);
    byte_copy((void**)&msg->endpoint, endpoint, tempLen + 1, 1);

    tempLen = strlen(username);
    byte_copy((void**)&msg->user, username, tempLen + 1, 1);

    tempLen = strlen(password);
    byte_copy((void**)&msg->password, password, tempLen + 1, 1);

    tempLen = strlen(topic);
    byte_copy((void**)&msg->topic, topic, tempLen + 1, 1);

    tempLen = payloadlen;
    byte_copy((void**)&msg->payload, payload, tempLen, 0);
    msg->payloadlen = payloadlen;

    msg->retain = retain;

    if (certfile == NULL) {
        msg->certfile = NULL;
    }
    else
    {
        tempLen = strlen(certfile);
        byte_copy((void**)&msg->certfile, certfile, tempLen + 1, 1);
    }


    MqttSender* sender = SENDERS[handle];
    Thread_lock_mutex(sender->lock);
    MqttMessageToPub* list = &(sender->incomingQueue);
    while (list->next != NULL)
    {
        list = list->next;
    }
    list->next = msg;
    Thread_unlock_mutex(sender->lock);
}
```
