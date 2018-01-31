# iot-edge-sdk-for-iot-parser ringbuffer

* 有两个线程，一个用于产生要发送的数据,保存在尾插队列中，一个用于将数据保存到文件或者发送；
* 数据保存在文件中，metadata是数据信息；

## Hacking Code

···
int main(int argc, char* argv[])
{
    init_and_start();                -----------+
                                                |
    wait_user_input();                          |
                                                |
    clean_and_exit();                           |
                                                |
    return 0;                                   |
}                                               |
                                                |
void init_and_start()                 <---------+
{
    printf("Baidu IoT Edge SDK v0.2.0\r\n");

    init_static_data();

    //
    // 1 load gateway configuration from local file
    // 2 receive device(slave) polling config from cloud, or local cache
    // 3 start execution, modbus read and PUB

    // 1 load gateway configuration from local file
    if (load_gateway_config(&g_gateway_conf))
    {
        printf("successfully loaded gateway config from file %s\r\n", CONFIG_FILE);
    }
    else
    {
        printf("failed to load gateway configuration from file %s\r\n", CONFIG_FILE);
    }

    g_mqttsender = new_mqtt_sender(DATA_CACHE, g_cache_size);             ------+
                                      ^---------------------------------------+ |
    // 2 receive device(slave) polling config from cloud, or local cache      | |
    g_slave_header.next = NULL;                                               | |
    load_slave_policy_from_cache(&g_slave_header);                            | |
                                                                              | |
    start_listen_command();                      -----------------------------*-*----+
    start_worker();                              -----------------------------*-*----*-------------------------+
}                                                                             | |    |                         |
                                                                              | |    |                         |
const char* const DATA_CACHE = "data_cache.dat";        <---------------------+ |    |                         |
                                                                                |    |                         |
int new_mqtt_sender(const char* cacheFile, int cacheSize)           <-----------+    |                         |
{                                                                                    |                         |
    if (lock_initialized == 0)                                                       |                         |
    {                                                                                |                         |
        sender_lock = Thread_create_mutex();                                         |                         |
        int j = 0;                                                                   |                         |
        for (; j < MAX_SENDER; j++)                                                  |                         |
        {                                                                            |                         |
            SENDERS[j] = NULL;                                                       |                         |
        }                                                                            |                         |
        lock_initialized = 1;                                                        |                         |
    }                                                                                |                         |
    RingBuFi* buf = newRingBuFi(cacheFile, (size_t) cacheSize);     ------+          |                         |
    MqttSender* sender = (MqttSender*) malloc(sizeof(MqttSender));        |          |                         |
    sender->lock = Thread_create_mutex();                                 |          |                         |
    sender->ringbuf = buf;                                                |          |                         |
    sender->mqttClients = NULL;                                           |          |                         |
    sender->badBrokers = NULL;                                            |          |                         |
    sender->status = WORKER_NOT_STARTED;                                  |          |                         |
    sender->worker = Thread_start(worker_func, (void*) sender);     ------*----------*-+                       |
    sender->status = WORKER_RUNNING;                                      |          | |                       |
    sender->incomingQueue.next = NULL;                                    |          | |                       |
    Thread_lock_mutex(sender_lock);                                       |          | |                       |
    int i = 0;                                                            |          | |                       |
    int ret = -1;                                                         |          | |                       |
    for (; i < MAX_SENDER; i++)                                           |          | |                       |
    {                                                                     |          | |                       |
        if (SENDERS[i] == NULL)                                           |          | |                       |
        {                                                                 |          | |                       |
            SENDERS[i] = sender;                                          |          | |                       |
            ret = i;                                                      |          | |                       |
            break;                                                        |          | |                       |
        }                                                                 |          | |                       |
    }                                                                     |          | |                       |
    Thread_unlock_mutex(sender_lock);                                     |          | |                       |
    return ret;                                                           |          | |                       |
}                                                                         |          | |                       |
                                                                          |          | |                       |
RingBuFi* newRingBuFi(const char* file, size_t sizeLimit) {     <---------+          | |                       |
                                                                                     | |                       |
    if (file == NULL) {                                                              | |                       |
        return NULL;                                                                 | |                       |
    }                                                                                | |                       |
    // check existence of the file                                                   | |                       |
    RingBuFi* pBuf = malloc(sizeof(RingBuFi));              --------------------+    | |                       |
    pBuf->sizeLimit = sizeLimit;                                                |    | |                       |
    FILE* fp = fopen(file, "r+b");                                              |    | |                       |
    if (fp == NULL) {                                                           |    | |                       |
        // file not exist, create it                                            |    | |                       |
        fp = fopen(file, "w+b");                                                |    | |                       |
        if (fp == NULL) {                                                       |    | |                       |
            // failed to create the file                                        |    | |                       |
            return NULL;                                                        |    | |                       |
        }                                                                       |    | |                       |
                                                                                |    | |                       |
        pBuf->fp = fp;                                                          |    | |                       |
        pBuf->sizeLimit = sizeLimit;                                            |    | |                       |
        pBuf->next = metaLen();                             ---------------+    |    | |                       |
        pBuf->head = pBuf->next;                                           |    |    | |                       |
        pBuf->nomanland = pBuf->next;                                      |    |    | |                       |
        pBuf->blockCnt = 0;                                                |    |    | |                       |
        // init the file                                                   |    |    | |                       |
        saveMeta(pBuf);                                     ---------------*-+  |    | |                       |
    } else {                                                               | |  |    | |                       |
        fread((void*)&pBuf->head, sizeof(pBuf->head), 1, fp);              | |  |    | |                       |
        fread((void*)&pBuf->next, sizeof(pBuf->next), 1, fp);              | |  |    | |                       |
        fread((void*)&pBuf->nomanland, sizeof(pBuf->nomanland), 1, fp);    | |  |    | |                       |
        fread((void*)&pBuf->blockCnt, sizeof(pBuf->blockCnt), 1, fp);      | |  |    | |                       |
        fread((void*)&pBuf->sizeLimit, sizeof(pBuf->sizeLimit), 1, fp);    | |  |    | |                       |
        pBuf->fp = fp;                                                     | |  |    | |                       |
    }                                                                      | |  |    | |                       |
    return pBuf;                                                           | |  |    | |                       |
}                                                                          | |  |    | |                       |
                                                                           | |  |    | |                       |
typedef struct {                                                           | |  |    | |                       |
    char* file;    // the file path                                        | |  |    | |                       |
    FILE* fp;    // the file handler                                       | |  |    | |                       |
    size_t blockCnt;    // number of records                               | |  |    | |                       |
    size_t sizeLimit;    // max file size can be used                      | |  |    | |                       |
    size_t next;    // file offset for the next record                     | |  |    | |                       |
    size_t head;        // file offset of the first (eldest) record        | |  |    | |                       |
    size_t nomanland;    // stop reading after this offset                 | |  |    | |                       |
} RingBuFi;                                            <-------------------*-*--+    | |                       |
                                                                           | |       | |                       |
size_t metaLen() {             <-------------------------------------------+ |       | |                       |
    /**                                                                    | |       | |                       |
     * typedef struct {                                                      |       | |                       |
     *     char* file;    // the file path                                   |       | |                       |
     *     FILE* fp;    // the file handler                                  |       | |                       |
     *     size_t blockCnt;    // number of records                          |       | |                       |
     *     size_t sizeLimit;    // max file size can be used                 |       | |                       |
     *     size_t next;    // file offset for the next record                |       | |                       |
     *     size_t head;        // file offset of the first (eldest) record   |       | |                       |
     *     size_t nomanland;    // stop reading after this offset            |       | |                       |
     * } RingBuFi;                                                           |       | |                       |
     * 这里的乘以5是因为只需要保存结构体后面的5个size_t类型的数据            |       | |
     */                                                                      |       | |                       |
    return sizeof(size_t) * 5;                                               |       | |                       |
}                                                                            |       | |                       |
                                                                             |       | |                       |
void saveMeta(RingBuFi* pBuf) {                        <---------------------+       | |                       |
    if (pBuf != NULL && pBuf->fp != NULL) {                                          | |                       |
        fseek(pBuf->fp, 0, SEEK_SET);                                                | |                       |
        fwrite((const void*)&pBuf->head, sizeof(pBuf->head), 1, pBuf->fp);           | |                       |
        fwrite((const void*)&pBuf->next, sizeof(pBuf->next), 1, pBuf->fp);           | |                       |
        fwrite((const void*)&pBuf->nomanland, sizeof(pBuf->nomanland), 1, pBuf->fp); | |                       |
        fwrite((const void*)&pBuf->blockCnt, sizeof(pBuf->blockCnt), 1, pBuf->fp);   | |                       |
        fwrite((const void*)&pBuf->sizeLimit, sizeof(pBuf->sizeLimit), 1, pBuf->fp); | |                       |
    }                                                                                | |                       |
}                                                                                    | |                       |
                                                                                     | |                       |
void start_listen_command()                            <-----------------------------+ |                       |
{                                                                                      |                       |
    printf("connecting gateway to cloud...\r\n");                                      |                       |
    Thread_lock_mutex(g_gateway_mutex);                                                |                       |
    // sub to config mqtt topic, to receive slave policy from cloud                    |                       |
    MQTTClient client;                                                                 |                       |
    MQTTClient_connectOptions conn_opts = MQTTClient_connectOptions_initializer;       |                       |
    int rc = 0;                                                                        |                       |
    int ch = 0;                                                                        |                       |
                                                                                       |                       |
    char clientid[MAX_LEN];                                                            |                       |
    snprintf(clientid, MAX_LEN, "modbusGW%lld", (long long)time(NULL));                |                       |
    MQTTClient_create(&client, g_gateway_conf.endpoint, clientid,                      |                       |
        MQTTCLIENT_PERSISTENCE_NONE, NULL);                                            |                       |
                                                                                       |                       |
    conn_opts.keepAliveInterval = 20;                                                  |                       |
    conn_opts.cleansession = 1;                                                        |                       |
    conn_opts.username = g_gateway_conf.user;                                          |                       |
    conn_opts.password = g_gateway_conf.password;                                      |                       |
    conn_opts.connectTimeout = 5;                                                      |                       |
    set_ssl_option(&conn_opts, g_gateway_conf.endpoint);                               |                       |
                                                                                       |                       |
    MQTTClient_setCallbacks(client, client, connection_lost, msg_arrived, delivered);  |                       |
                                                      ^-------------------------+      |                       |
    if ((rc = MQTTClient_connect(client, &conn_opts)) != MQTTCLIENT_SUCCESS)    |      |                       |
    {                                                                           |      |                       |
        MQTTClient_destroy(&client);                                            |      |                       |
        printf("Failed to connect, return code %d\r\n", rc);                    |      |                       |
        Thread_unlock_mutex(g_gateway_mutex);                                   |      |                       |
                                                                                |      |                       |
        return;                                                                 |      |                       |
    }                                                                           |      |                       |
    if (strlen(g_gateway_conf.backControlTopic) > 0) {                          |      |                       |
        char* topics[2];                                                        |      |                       |
        topics[0] = g_gateway_conf.topic;                                       |      |                       |
        topics[1] = g_gateway_conf.backControlTopic;                            |      |                       |
        int qoss[2];                                                            |      |                       |
        qoss[0] = 0;                                                            |      |                       |
        qoss[1] = 0;                                                            |      |                       |
        MQTTClient_subscribeMany(client, 2, topics, qoss);                      |      |                       |
    } else {                                                                    |      |                       |
        MQTTClient_subscribe(client, g_gateway_conf.topic, 0);                  |      |                       |
    }                                                                           |      |                       |
                                                                                |      |                       |
    g_gateway_connected = 1;                                                    |      |                       |
    Thread_unlock_mutex(g_gateway_mutex);                                       |      |                       |
}                                                                               |      |                       |
                                                                                |      |                       |
void connection_lost(void* context, char* cause)                <---------------+      |                       |
{                                                                                      |                       |
    printf("\nConnection lost, caused by %s, will reconnect later\r\n", cause);        |                       |
    MQTTClient client = (MQTTClient) context;                                          |                       |
    MQTTClient_destroy(&client);                                                       |                       |
    Thread_lock_mutex(g_gateway_mutex);                                                |                       |
    g_gateway_connected = 0;                                                           |                       |
    Thread_unlock_mutex(g_gateway_mutex);                                              |                       |
}                                                                                      |                       |
                                                                                       |                       |
static thread_return_type worker_func(void* arg)                 <---------------------+                       |
{                                                                                                              |
    // peek a record if any                                                                                    |
    // establish mqtt connection if need                                                                       |
    // send the record and remove from buf                                                                     |
    // sender->worker = Thread_start(worker_func, (void*) sender);                                             |
    sender->status = WORKER_RUNNING;                                                                           |
    MqttSender* sender = (MqttSender*) arg;                                                                    |
    MqttMessageToPub sendingQueue;                                                                             |
    sendingQueue.next = NULL;                                                                                  |
    MqttMessageToPub* msg = NULL;                                                                              |
    while (sender != NULL && sender->status != WORKER_REQUEST_STOP)                                            |
    {                                                                                                          |
        if (sendingQueue.next != NULL)                                                                         |
        {                                                                                                      |
            // 1, save incoming queue into file                                                                |
            flushIncomingQueueToFile(sender);                                --------------------------------+ |
        }                                                                                                    | |
        else                                                                                                 | |
        {                                                                                                    | |
            // if buffer is not empty, save incoming queue to buffer                                         | |
            // and fetch data from buffer                                                                    | |
            // otherwise try to get from incoming queue                                                      | |
            if (! isRingBuFiEmpty(sender->ringbuf))                                                          | |
            {                                                                                                | |
                flushIncomingQueueToFile(sender);                                                            | |
                                                                                                             | |
                // fetch one record from buffer                                                              | |
                void* data = NULL;                                                                           | |
                size_t len = 0;                                                                              | |
                peekRingBuFiRecord(sender->ringbuf, &data, &len);                                            | |
                popRingBuFiRecord(sender->ringbuf);                                                          | |
                if (data != NULL && len > 0)                                                                 | |
                {                                                                                            | |
                    sendingQueue.next = deserializeMsg(data);                                                | |
                    free(data);                                                                              | |
                }                                                                                            | |
            }                                                                                                | |
            else                                                                                             | |
            {                                                                                                | |
                // get data from incoming queue directory                                                    | |
                if (sender->incomingQueue.next != NULL)                                                      | |
                {                                                                                            | |
                    Thread_lock_mutex(sender->lock);                                                         | |
                    sendingQueue.next = sender->incomingQueue.next;                                          | |
                    sender->incomingQueue.next = NULL;                                                       | |
                    Thread_unlock_mutex(sender->lock);                                                       | |
                }                                                                                            | |
            }                                                                                                | |
        }                                                                                                    | |
                                                                                                             | |
        if (sendingQueue.next == NULL)                                                                       | |
        {                                                                                                    | |
            // no date to send, sleep for a while                                                            | |
            sleep(1);                                                                                        | |
        }                                                                                                    | |
        else                                                                                                 | |
        {                                                                                                    | |
            msg = sendingQueue.next;                                                                         | |
            // it's known bad broker?                                                                        | |
            if (isKnownBadBroker(sender, msg->endpoint, msg->user, msg->password))                           | |
            {                                                                                                | |
                // discord this message and continue                                                         | |
                sendingQueue.next = sendingQueue.next->next;                                                 | |
                freeMsg(msg);                                                                                | |
                printf("got msg of an known bad broker\n");                                                  | |
                continue;                                                                                    | |
            }                                                                                                | |
                                                                                                             | |
            MQTTClient mqttClient = findExistingClient(sender, msg->endpoint, msg->user);                    | |
            if (mqttClient == NULL)                                                                          | |
            {                                                                                                | |
                // let's make a connection                                                                   | |
                MQTTClient newClient;                                                                        | |
                int rc = makeMqttConnection(&newClient, msg->endpoint, msg->user, msg->password,             | |
                 msg->certfile);                                                                             | |
                                                                                                             | |
                if (rc != MQTTCLIENT_SUCCESS)                                                                | |
                {                                                                                            | |
                    MQTTClient_destroy(&newClient);                                                          | |
                    if (rc == 1     // Unacceptable protocol version                                         | |
                        || rc == 4    // Bad user name or password                                           | |
                        || rc == 5)    // Not authorized                                                     | |
                    {                                                                                        | |
                        MqttBrokerId* badBroker = (MqttBrokerId*) malloc(sizeof(MqttBrokerId));              | |
                        byte_copy((void**)&badBroker->endpoint, msg->endpoint, strlen(msg->endpoint) + 1, 1);| |
                        byte_copy((void**)&badBroker->user, msg->user, strlen(msg->user) + 1, 1);            | |
                        byte_copy((void**)&badBroker->password, msg->password, strlen(msg->password) + 1, 1);| |
                        badBroker->next = sender->badBrokers;                                                | |
                        sender->badBrokers = badBroker;                                                      | |
                        printf("Found a bad broker\n");                                                      | |
                    }                                                                                        | |
                    sleep(1);                                                                                | |
                    continue;                                                                                | |
                }                                                                                            | |
                                                                                                             | |
                mqttClient = newClient;                                                                      | |
                MqttBrokerId* broker = (MqttBrokerId *) malloc(sizeof(MqttBrokerId));                        | |
                byte_copy((void**)&broker->endpoint, msg->endpoint, strlen(msg->endpoint) + 1, 1);           | |
                byte_copy((void**)&broker->user, msg->user, strlen(msg->user) + 1, 1);                       | |
                byte_copy((void**)&broker->password, msg->password, strlen(msg->password) + 1, 1);           | |
                                                                                                             | |
                broker->client = mqttClient;                                                                 | |
                broker->next = sender->mqttClients;                                                          | |
                sender->mqttClients = broker;                                                                | |
            }                                                                                                | |
                                                                                                             | |
            MQTTClient_message pubmsg = MQTTClient_message_initializer;                                      | |
            MQTTClient_deliveryToken delivery_token;                                                         | |
                                                                                                             | |
            pubmsg.payload = msg->payload;                                                                   | |
            pubmsg.payloadlen = msg->payloadlen;                                                             | |
            pubmsg.qos = 1;                                                                                  | |
            pubmsg.retained = msg->retain;                                                                   | |
                                                                                                             | |
            int rc = MQTTClient_publishMessage(mqttClient,                                                   | |
                     msg->topic, &pubmsg, &delivery_token);                                                  | |
                                                                                                             | |
            if (rc == MQTTCLIENT_SUCCESS)                                                                    | |
            {                                                                                                | |
                int waitrc = MQTTClient_waitForCompletion(mqttClient, delivery_token, 10000L);               | |
                if (waitrc == MQTTCLIENT_SUCCESS)                                                            | |
                {                                                                                            | |
                    sendingQueue.next = sendingQueue.next->next;                                             | |
                    freeMsg(msg);                                                                            | |
                }                                                                                            | |
            }                                                                                                | |
            else                                                                                             | |
            {                                                                                                | |
                removeExistingClient(sender, mqttClient);                                                    | |
                MQTTClient_disconnect(mqttClient, 3000);                                                     | |
                MQTTClient_destroy(&mqttClient);                                                             | |
                sleep(1);                                                                                    | |
            }                                                                                                | |
        }                                                                                                    | |
                                                                                                             | |
    }                                                                                                        | |
    sender->status = WORKER_STOPPED;                                                                         | |
}                                                                                                            | |
                                                                                                             | |
                                                                                                             | |
/**                                                                                                          | |
 * typedef struct                                                                                            | |
 * {                                                                                                         | |
 *     mutex_type lock;                                                                                      | |
 *     RingBuFi* ringbuf;                                                                                    | |
 *     MqttBrokerId* mqttClients;                                                                            | |
 *     MqttBrokerId* badBrokers;                                                                             | |
 *     volatile char status;                                                                                 | |
 *     thread_type worker;                                                                                   | |
 *     MqttMessageToPub incomingQueue;    // just the header, real msg start from next                       | |
 * } MqttSender;                                                                                             | |
 */                                                                                                          | |
static flushIncomingQueueToFile(MqttSender* sender)                         <--------------------------------+ |
{                                                                                                              |
    if (sender != NULL && sender->incomingQueue.next != NULL)                                                  |
    {                                                                                                          |
        Thread_lock_mutex(sender->lock);                                                                       |
        MqttMessageToPub* tosave = sender->incomingQueue.next;              -----------------------------------*-+
        sender->incomingQueue.next = NULL;                                                                     | |
        Thread_unlock_mutex(sender->lock);                                                                     | |
        while (tosave != NULL)                                                                                 | |
        {                                                                                                      | |
            void* bytes;                                                                                       | |
            size_t msg_len = serializeMsg(tosave, &bytes);       --------+                                     | |
            if (bytes != NULL && msg_len > 0)                            |                                     | |
            {                                                            |                                     | |
                putRingBuFiRecord(sender->ringbuf, bytes, msg_len);  ----*-+                                   | |
                free(bytes);                                             | |                                   | |
            }                                                            | |                                   | |
            MqttMessageToPub* todel = tosave;                            | |                                   | |
            tosave = tosave->next;                                       | |                                   | |
            freeMsg(todel);                                              | |                                   | |
        }                                                                | |                                   | |
    }                                                                    | |                                   | |
}                                                                        | |                                   | |
                                                                         | |                                   | |
size_t serializeMsg(const MqttMessageToPub* msg, void** output)  <-------+ |                                   | |
{                                ^---------------------------------------+ |                                   | |
    size_t len = messageLen(msg);                ----------------------+ | |                                   | |
    if (len <= 0) {                                                    | | |                                   | |
        return 0;                                                      | | |                                   | |
    }                                                                  | | |                                   | |
                                                                       | | |                                   | |
    *output = malloc(len);                                             | | |                                   | |
    size_t idx = 0;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // endpoint                                                        | | |                                   | |
    size_t tempLen = strlen(msg->endpoint);                            | | |                                   | |
    memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));           | | |                                   | |
    idx += sizeof(size_t);                                             | | |                                   | |
    memcpy(*output + idx, msg->endpoint, tempLen);                     | | |                                   | |
    idx += tempLen;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // user                                                            | | |                                   | |
    tempLen = strlen(msg->user);                                       | | |                                   | |
    memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));           | | |                                   | |
    idx += sizeof(size_t);                                             | | |                                   | |
    memcpy(*output + idx, msg->user, tempLen);                         | | |                                   | |
    idx += tempLen;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // password                                                        | | |                                   | |
    tempLen = strlen(msg->password);                                   | | |                                   | |
    memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));           | | |                                   | |
    idx += sizeof(size_t);                                             | | |                                   | |
    memcpy(*output + idx, msg->password, tempLen);                     | | |                                   | |
    idx += tempLen;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // topic                                                           | | |                                   | |
    tempLen = strlen(msg->topic);                                      | | |                                   | |
    memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));           | | |                                   | |
    idx += sizeof(size_t);                                             | | |                                   | |
    memcpy(*output + idx, msg->topic, tempLen);                        | | |                                   | |
    idx += tempLen;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // retain                                                          | | |                                   | |
    memcpy(*output + idx, (void*) &msg->retain, sizeof(char));         | | |                                   | |
    idx += sizeof(char);                                               | | |                                   | |
                                                                       | | |                                   | |
    // payload                                                         | | |                                   | |
    tempLen = msg->payloadlen;                                         | | |                                   | |
    memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));           | | |                                   | |
    idx += sizeof(size_t);                                             | | |                                   | |
    memcpy(*output + idx, msg->payload, tempLen);                      | | |                                   | |
    idx += tempLen;                                                    | | |                                   | |
                                                                       | | |                                   | |
    // certfile                                                        | | |                                   | |
    if (msg->certfile != NULL)                                         | | |                                   | |
    {                                                                  | | |                                   | |
        tempLen = strlen(msg->certfile);                               | | |                                   | |
        memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));       | | |                                   | |
        idx += sizeof(size_t);                                         | | |                                   | |
        memcpy(*output + idx, msg->certfile, tempLen);                 | | |                                   | |
        idx += tempLen;                                                | | |                                   | |
    }                                                                  | | |                                   | |
    else                                                               | | |                                   | |
    {                                                                  | | |                                   | |
        tempLen = 0;                                                   | | |                                   | |
        memcpy(*output + idx, (void*) &tempLen, sizeof(size_t));       | | |                                   | |
        idx += sizeof(size_t);                                         | | |                                   | |
    }                                                                  | | |                                   | |
    return idx;                                                        | | |                                   | |
}                                                                      | | |                                   | |
                                                                       | | |                                   | |
typedef struct MqttMessageToPub_t                 <----------------------+ |                                   | |
{                                                                      |   |                                   | |
    char* endpoint;                                                    |   |                                   | |
    char* user;                                                        |   |                                   | |
    char* password;                                                    |   |                                   | |
    char* topic;                                                       |   |                                   | |
    char retain;                                                       |   |                                   | |
    char* payload;                                                     |   |                                   | |
    int payloadlen;                                                    |   |                                   | |
    char* certfile;                                                    |   |                                   | |
    struct MqttMessageToPub_t* next;                                   |   |                                   | |
} MqttMessageToPub;                                                    |   |                                   | |
                                                                       |   |                                   | |
size_t messageLen(const MqttMessageToPub* msg)          <--------------+   |                                   | |
{                                                                          |                                   | |
    size_t len = 0;                                                        |                                   | |
    if (msg == NULL)                                                       |                                   | |
    {                                                                      |                                   | |
        return len;                                                        |                                   | |
    }                                                                      |                                   | |
                                                                           |                                   | |
    len += sizeof(size_t) + strlen(msg->endpoint);                         |                                   | |
    len += sizeof(size_t) + strlen(msg->user);                             |                                   | |
    len += sizeof(size_t) + strlen(msg->password);                         |                                   | |
    len += sizeof(size_t) + strlen(msg->topic);                            |                                   | |
    len += sizeof(char);                                                   |                                   | |
    len += sizeof(size_t) + msg->payloadlen;                               |                                   | |
    len += sizeof(size_t);                                                 |                                   | |
    if (msg->certfile != NULL)                                             |                                   | |
    {                                                                      |                                   | |
        len += strlen(msg->certfile);                                      |                                   | |
    }                                                                      |                                   | |
                                                                           |                                   | |
    return len;                                                            |                                   | |
}                                                                          |                                   | |
                                                                           |                                   | |
int putRingBuFiRecord(RingBuFi* pBuf, const void* pBytes, size_t len) { <--+                                   | |
    if (pBuf == NULL || pBuf->fp == NULL || pBytes == NULL || len <= 0) {                                      | |
        return 0;                                                                                              | |
    }                                                                                                          | |
    if (pBuf->sizeLimit - metaLen() < len + sizeof(len))                                                       | |
    {                                                                                                          | |
        return 0;     // limit is too small                                                                    | |
    }                                                                                                          | |
                                                                                                               | |
    size_t sizeToWrite = sizeof(size_t) + len;                                                                 | |
    if (pBuf->blockCnt <= 0)                                                                                   | |
    {                                                                                                          | |
        // the buf is empty, start from the beginning                                                          | |
        pBuf->next = metaLen();                                                                                | |
        pBuf->head = pBuf->next;                                                                               | |
        writeBytesAt(pBuf->fp, pBuf->next, (const void*) &len, sizeof(len));                                   | |
        writeBytesAt(pBuf->fp, pBuf->next + sizeof(len), pBytes, len);                                         | |
        pBuf->blockCnt = 1;                                                                                    | |
        pBuf->nomanland = pBuf->next + sizeToWrite;                                                            | |
        pBuf->next += sizeToWrite;                                                                             | |
        saveMeta(pBuf);                                                                                        | |
        // truncate the file to minimize the disk usage                                                        | |
        if (ftruncate(fileno(pBuf->fp), pBuf->nomanland) != 0)                                                 | |
        {                                                                                                      | |
            printf("[WARN] failed to shorten the file size by ftruncate, no functional impact\r\n");           | |
        }                                                                                                      | |
    }                                                                                                          | |
    else if (pBuf->next > pBuf->head) {                                                                        | |
        if (pBuf->next + sizeToWrite > pBuf->sizeLimit) {                                                      | |
            // exceeded the sizeLimit, write to the start of data block                                        | |
            // override record(s) at the beginning of data block if needed                                     | |
            size_t bytesFreed = pBuf->head - metaLen();                                                        | |
            bytesFreed = eraseEnoughSpaceOrEof(pBuf, bytesFreed, sizeToWrite);                                 | |
                                                                                                               | |
            writeBytesAt(pBuf->fp, metaLen(), (const void*) &len, sizeof(len));                                | |
            writeBytesAt(pBuf->fp, metaLen() + sizeof(size_t), pBytes, len);                                   | |
            increaseBlockCnt(pBuf, 1); // pBuf->blockCnt++;                                                    | |
            pBuf->nomanland = pBuf->next;                                                                      | |
            if (pBuf->head >= pBuf->nomanland)                                                                 | |
            {                                                                                                  | |
                pBuf->head = metaLen();                                                                        | |
            }                                                                                                  | |
            pBuf->next = sizeToWrite + metaLen();                                                              | |
            saveMeta(pBuf);                                                                                    | |
        } else {                                                                                               | |
            // just put the block to the end of file                                                           | |
            size_t newNext = pBuf->next + sizeToWrite;                                                         | |
            writeBytesAt(pBuf->fp, pBuf->next, (const void*) &len, sizeof(len));                               | |
            size_t byteWritten = writeBytesAt(pBuf->fp, pBuf->next + sizeof(len), pBytes, len);                | |
            increaseBlockCnt(pBuf, 1); // pBuf->blockCnt++;                                                    | |
            pBuf->next = newNext;                                                                              | |
            pBuf->nomanland = pBuf->next;                                                                      | |
            saveMeta(pBuf);                                                                                    | |
        }                                                                                                      | |
                                                                                                               | |
    } else { // if (pBuf->next <= pBuf->head) {                                                                | |
                                                                                                               | |
        if (pBuf->head - pBuf->next >= sizeToWrite) {                                                          | |
            // there is enough space between head and next                                                     | |
        } else {                                                                                               | |
            if (pBuf->nomanland - pBuf->next >= sizeToWrite)                                                   | |
            {                                                                                                  | |
                // erase existing records should be enough                                                     | |
                size_t bytesFreed = pBuf->head - pBuf->next;                                                   | |
                bytesFreed = eraseEnoughSpaceOrEof(pBuf, bytesFreed, sizeToWrite);                             | |
                if (pBuf->head >= pBuf->nomanland)                                                             | |
                {                                                                                              | |
                    pBuf->head = metaLen();                                                                    | |
                }                                                                                              | |
            }                                                                                                  | |
            else if (pBuf->sizeLimit - pBuf->next >= sizeToWrite)                                              | |
            {                                                                                                  | |
                // all the records after head will be erased,                                                  | |
                // erase all the records after head                                                            | |
                eraseEnoughSpaceOrEof(pBuf, 0, pBuf->sizeLimit);                                               | |
                pBuf->head = metaLen();                                                                        | |
                pBuf->nomanland = pBuf->next;                                                                  | |
            }                                                                                                  | |
            else                                                                                               | |
            {                                                                                                  | |
                // erase from the begining                                                                     | |
                if (pBuf->next - metaLen() >= sizeToWrite)                                                     | |
                {                                                                                              | |
                    // erase from the beinning                                                                 | |
                    pBuf->nomanland = pBuf->next;                                                              | |
                    pBuf->head = metaLen();                                                                    | |
                                                                                                               | |
                    size_t bytesFreed = eraseEnoughSpaceOrEof(pBuf, 0, sizeToWrite);                           | |
                }                                                                                              | |
                else                                                                                           | |
                {                                                                                              | |
                    // every thing will be erased                                                              | |
                    pBuf->next = metaLen();                                                                    | |
                    pBuf->head = pBuf->next;                                                                   | |
                    pBuf->nomanland = pBuf->next;                                                              | |
                    pBuf->blockCnt = 0;                                                                        | |
                }                                                                                              | |
            }                                                                                                  | |
        }                                                                                                      | |
        writeBytesAt(pBuf->fp, pBuf->next, (const void*) &len, sizeof(len));                                   | |
        writeBytesAt(pBuf->fp, pBuf->next + sizeof(size_t), pBytes, len);                                      | |
        increaseBlockCnt(pBuf, 1);                                                                             | |
        pBuf->next += sizeToWrite;                                                                             | |
        if (pBuf->next > pBuf->head)                                                                           | |
        {                                                                                                      | |
            pBuf->nomanland = pBuf->next;                                                                      | |
        }                                                                                                      | |
        saveMeta(pBuf);                                                                                        | |
    }                                                                                                          | |
                                                                                                               | |
    return len;                                                                                                | |
}                                                                                                              | |
                                                                                                               | |
void start_worker()                                        <---------------------------------------------------+ |
{                                                                                                                |
    g_worker_thread = Thread_start(worker_func, (void*) NULL);           -------+                                |
}                                                                               |                                |
                                                                                |                                |
//TODO: fire up a few more workers, and precess in parallel, to speed up.       |                                |
static thread_return_type worker_func(void* arg)                         <------+                                |
{                                                                                                                |
    g_worker_is_running = 1;                                                                                     |
    while (g_stop_worker != 1)                                                                                   |
    {                                                                                                            |
        // load slave policy if it's updated                                                                     |
        if (g_policy_updated)                                                                                    |
        {                                                                                                        |
            load_slave_policy_from_cache(&g_slave_header);                                                       |
        }                                                                                                        |
                                                                                                                 |
        if (g_gateway_connected == 0)                                                                            |
        {                                                                                                        |
            start_listen_command();                                                                              |
        }                                                                                                        |
                                                                                                                 |
        // iterate from the beginning of the policy list                                                         |
        // and pick those whose nextRun is due, and execute them,                                                |
        // calculate the new next run, and insert into the list                                                  |
        time_t now = time(NULL);                                                                                 |
        // we have something to do, acquire the lock here                                                        |
        int rc = Thread_lock_mutex(g_policy_lock);                                                               |
        if (g_slave_header.next != NULL && g_slave_header.next->nextRun <= now)                                  |
        {                                                                                                        |
            while (g_slave_header.next != NULL && g_slave_header.next->nextRun <= now)                           |
            {                                                                                                    |
                // detach from the list                                                                          |
                SlavePolicy* policy = g_slave_header.next;                                                       |
                g_slave_header.next = g_slave_header.next->next;                                                 |
                execute_policy(policy);          --------+                                                       |
            }                                            |                                                       |
        }                                                |                                                       |
        rc = Thread_unlock_mutex(g_policy_lock);         |                                                       |
                                                         |                                                       |
        sleep(1);                                        |                                                       |
    }                                                    |                                                       |
    log_debug("exiting worker thread...\r\n");           |                                                       |
    g_worker_is_running = 0;                             |                                                       |
}                                                        |                                                       |
                                                         |                                                       |
void execute_policy(SlavePolicy* policy)        <--------+                                                       |
{                                                                                                                |
    // 3 recaculate the next run time                                                                            |
    policy->nextRun = policy->interval + time(NULL);                                                             |
                                                                                                                 |
    // 4 re-insert into the list, in order of nextRun                                                            |
    insert_slave_policy(policy);                                                                                 |
                                                                                                                 |
    if (policy == NULL)                                                                                          |
    {                                                                                                            |
        return;                                                                                                  |
    }                                                                                                            |
                                                                                                                 |
    // 1 query modbus data                                                                                       |
    char payload[1024];                                                                                          |
    read_modbus(policy, payload);                                                                                |
    // 2 pub modbus data                                                                                         |
    if (strlen(payload) > 0)                                                                                     |
    {                                                                                                            |
        char msgcontent[BUFF_LEN];                                                                               |
        pack_pub_msg(policy, payload, msgcontent);                                                               |
        mqtt_send(g_mqttsender,                  ------------+                                                   |
                    policy->pubChannel.endpoint,             |                                                   |
                    policy->pubChannel.user,                 |                                                   |
                    policy->pubChannel.password,             |                                                   |
                    policy->pubChannel.topic,                |                                                   |
                    msgcontent,                              |                                                   |
                    strlen(msgcontent),                      |                                                   |
                    0,                                       |                                                   |
                    PEM_FILE);                               |                                                   |
    }                                                        |                                                   |
}                                                            |                                                   |
                                                             |                                                   |
char mqtt_send(int handle,                  <----------------+                                                   |
    const char* endpoint,                                                                                        |
    const char* username,                                                                                        |
    const char* password,                                                                                        |
    const char* topic,                                                                                           |
    const char* payload,                                                                                         |
    int payloadlen,                                                                                              |
    char retain,                                                                                                 |
    const char* certfile)                                                                                        |
{                                                                                                                |
    // 0, check if handle is valid or not                                                                        |
    if (handle < 0 || handle >= MAX_SENDER || SENDERS[handle] == NULL)                                           |
    {                                                                                                            |
        return -1;                                                                                               |
    }                                                                                                            |
                                                                                                                 |
    // 1, check if it's a bad broker                                                                             |
    if (isKnownBadBroker(SENDERS[handle], endpoint, username, password) == 1) {                                  |
        return -1;                                                                                               |
    }                                                                                                            |
                                                                                                                 |
    // add to ring buffer                                                                                        |
    MqttMessageToPub* msg = (MqttMessageToPub*) malloc(sizeof(MqttMessageToPub));                                |
    msg->next = NULL;                                                                                            |
    int tempLen = strlen(endpoint);                                                                              |
    byte_copy((void**)&msg->endpoint, endpoint, tempLen + 1, 1);                                                 |
                                                                                                                 |
    tempLen = strlen(username);                                                                                  |
    byte_copy((void**)&msg->user, username, tempLen + 1, 1);                                                     |
                                                                                                                 |
    tempLen = strlen(password);                                                                                  |
    byte_copy((void**)&msg->password, password, tempLen + 1, 1);                                                 |
                                                                                                                 |
    tempLen = strlen(topic);                                                                                     |
    byte_copy((void**)&msg->topic, topic, tempLen + 1, 1);                                                       |
                                                                                                                 |
    tempLen = payloadlen;                                                                                        |
    byte_copy((void**)&msg->payload, payload, tempLen, 0);                                                       |
    msg->payloadlen = payloadlen;                                                                                |
                                                                                                                 |
    msg->retain = retain;                                                                                        |
                                                                                                                 |
    if (certfile == NULL) {                                                                                      |
        msg->certfile = NULL;                                                                                    |
    }                                                                                                            |
    else                                                                                                         |
    {                                                                                                            |
        tempLen = strlen(certfile);                                                                              |
        byte_copy((void**)&msg->certfile, certfile, tempLen + 1, 1);                                             |
    }                                                                                                            |
                                                                                                                 |
                                                                                                                 |
    /**                                                                                                          |
     * typedef struct                                                                                            |
     * {                                                                                                         |
     *     mutex_type lock;                                                                                      |
     *     RingBuFi* ringbuf;                                                                                    |
     *     MqttBrokerId* mqttClients;                                                                            |
     *     MqttBrokerId* badBrokers;                                                                             |
     *     volatile char status;                                                                                 |
     *     thread_type worker;                                                                                   |
     *     MqttMessageToPub incomingQueue;    // just the header, real msg start from next                       |
     * } MqttSender;                                                                                             |
     */                                                                                                          |
    MqttSender* sender = SENDERS[handle];                                                                        |
    Thread_lock_mutex(sender->lock);                                                                             |
    MqttMessageToPub* list = &(sender->incomingQueue);                       <-----------------------------------+
    while (list->next != NULL)
    {
        list = list->next;
    }
    list->next = msg;
    Thread_unlock_mutex(sender->lock);
}
```
