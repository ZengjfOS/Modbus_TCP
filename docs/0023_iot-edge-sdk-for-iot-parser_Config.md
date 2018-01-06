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
