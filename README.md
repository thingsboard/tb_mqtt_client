# ThingsBoard MQTT client Python SDK
[![Join the chat at https://gitter.im/thingsboard/chat](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/thingsboard/chat?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<a href="https://thingsboard.io"><img src="./logo.png?raw=true" width="100" height="100"></a>

ThingsBoard is an open-source IoT platform for data collection, processing, visualization, and device management.
This project ia a Python library that provides convenient client SDK for both [Device](https://thingsboard.io/docs/reference/mqtt-api/) 
and [Gateway](https://thingsboard.io/docs/reference/gateway-mqtt-api/) APIs.

SDK supports:
- Unencrypted and encrypted (TLS v1.2) connection;
- QoS 0 and 1;
- Automatic reconnect;
- All Device MQTT APIs provided by ThingsBoard
- All Gateway MQTT APIs provided by ThingsBoard

SDK is based on Paho MQTT library. 

## Installation

To install using pip:

```bash
pip3 install tb-mqtt-client
```

## Getting Started

Client initialization and telemetry publishing

```python
from tb_device_mqtt import TBDeviceMqttClient, TBPublishInfo
telemetry = {"temperature": 41.9, "enabled": False, "currentFirmwareVersion": "v1.2.2"}
client = TBDeviceMqttClient("127.0.0.1", "A1_TEST_TOKEN")
# Connect to ThingsBoard
client.connect()
# Sending telemetry without checking the delivery status
client.send_telemetry(telemetry) 
# Sending telemetry and checking the delivery status (QoS = 1 by default)
result = client.send_telemetry(telemetry)
# get is a blocking call that awaits delivery status  
success = result.get() == TBPublishInfo.TB_ERR_SUCCESS
# Disconnect from ThingsBoard
client.disconnect()
```

### Connection using TLS

TLS connection to localhost. See https://thingsboard.io/docs/user-guide/mqtt-over-ssl/ for more information about client and ThingsBoard configuration.

```python
from tb_device_mqtt import TBDeviceMqttClient
import socket
client = TBDeviceMqttClient(socket.gethostname())
client.connect(tls=True,
               ca_certs="mqttserver.pub.pem",
               cert_file="mqttclient.nopass.pem")
client.disconnect()
```

## Using Device APIs

**TBDeviceMqttClient** provides access to Device MQTT APIs of ThingsBoard platform. It allows to publish telemetry and attribute updates, subscribe to attribute changes, send and receive RPC commands, etc. 
#### Subscribtion to attributes
```python
import time
from tb_device_mqtt import TBDeviceMqttClient

def callback(result):
    print(result)

client = TBDeviceMqttClient("127.0.0.1", "A1_TEST_TOKEN")
client.connect()
client.subscribe_to_attribute("uploadFrequency", callback)
client.subscribe_to_all_attributes(callback)
while True:
    time.sleep(1)
```

#### Telemetry pack sending
```python
import logging
from tb_device_mqtt import TBDeviceMqttClient, TBPublishInfo
import time
telemetry_with_ts = {"ts": int(round(time.time() * 1000)), "values": {"temperature": 42.1, "humidity": 70}}
client = TBDeviceMqttClient("127.0.0.1", "A1_TEST_TOKEN")
# we set maximum amount of messages sent to send them at the same time. it may stress memory but increases performance
client.max_inflight_messages_set(100)
client.connect()
results = []
result = True
for i in range(0, 100):
    results.append(client.send_telemetry(telemetry_with_ts))
for tmp_result in results:
    result &= tmp_result.get() == TBPublishInfo.TB_ERR_SUCCESS
print("Result " + str(result))
client.disconnect()
```

#### Request attributes from server
```python
import logging
import time
from tb_device_mqtt import TBDeviceMqttClient

def on_attributes_change(result, exception):
    if exception is not None:
        print("Exception: " + str(exception))
    else:
        print(result)

client = TBDeviceMqttClient("127.0.0.1", "A1_TEST_TOKEN")
client.connect()
client.request_attributes(["configuration","targetFirmwareVersion"], callback=on_attributes_change)
while True:
    time.sleep(1)
```
#### Respond to server RPC call
```python
import psutil
import time
import logging
from tb_device_mqtt import TBDeviceMqttClient

# dependently of request method we send different data back
def on_server_side_rpc_request(request_id, request_body):
    print(request_id, request_body)
    if request_body["method"] == "getCPULoad":
        client.send_rpc_reply(request_id, {"CPU percent": psutil.cpu_percent()})
    elif request_body["method"] == "getMemoryUsage":
        client.send_rpc_reply(request_id, {"Memory": psutil.virtual_memory().percent})

client = TBDeviceMqttClient("127.0.0.1", "A1_TEST_TOKEN")
client.set_server_side_rpc_request_handler(on_server_side_rpc_request)
client.connect()
while True:
    time.sleep(1)
```
## Using Gateway APIs

**TBGatewayMqttClient** extends **TBDeviceMqttClient**, thus has access to all it's APIs as a regular device.
Besides, gateway is able to represent multiple devices connected to it. For example, sending telemetry or attributes on behalf of other, constrained, device. See more info about the gateway here: 
#### Telemetry and attributes sending 
```python
import time
from tb_gateway_mqtt import TBGatewayMqttClient
gateway = TBGatewayMqttClient("127.0.0.1", "GATEWAY_TEST_TOKEN")
gateway.connect()
gateway.gw_connect_device("Test Device A1")

gateway.gw_send_telemetry("Test Device A1", {"ts": int(round(time.time() * 1000)), "values": {"temperature": 42.2}})
gateway.gw_send_attributes("Test Device A1", {"firmwareVersion": "2.3.1"})

gateway.gw_disconnect_device("Test Device A1")
gateway.disconnect()
```
#### Request attributes
```python
import logging
import time
from tb_gateway_mqtt import TBGatewayMqttClient

def callback(result, exception):
    if exception is not None:
        print("Exception: " + str(exception))
    else:
        print(result)

gateway = TBGatewayMqttClient("127.0.0.1", "TEST_GATEWAY_TOKEN")
gateway.connect()
gateway.gw_request_shared_attributes("Test Device A1", ["temperature"], callback)

while True:
    time.sleep(1)
```
#### Respond to RPC
```python
import time

from tb_gateway_mqtt import TBGatewayMqttClient
import psutil

def rpc_request_response(request_body):
    # request body contains id, method and other parameters
    print(request_body)
    method = request_body["data"]["method"]
    device = request_body["device"]
    req_id = request_body["data"]["id"]
    # dependently of request method we send different data back
    if method == 'getCPULoad':
        gateway.gw_send_rpc_reply(device, req_id, {"CPU load": psutil.cpu_percent()})
    elif method == 'getMemoryLoad':
        gateway.gw_send_rpc_reply(device, req_id, {"Memory": psutil.virtual_memory().percent})
    else:
        print('Unknown method: ' + method)

gateway = TBGatewayMqttClient("127.0.0.1", "TEST_GATEWAY_TOKEN")
gateway.connect()
# now rpc_request_response will process rpc requests from servers
gateway.gw_set_server_side_rpc_request_handler(rpc_request_response)
# without device connection it is impossible to get any messages
gateway.gw_connect_device("Test Device A1")
while True:
    time.sleep(1)
```
## Other Examples

There are more examples for both [device](https://github.com/thingsboard/thingsboard-python-client-sdk/tree/master/examples/device) and [gateway](https://github.com/thingsboard/thingsboard-python-client-sdk/tree/master/examples/gateway) in corresponding [folders](https://github.com/thingsboard/thingsboard-python-client-sdk/tree/master/examples).

## Support

 - [Community chat](https://gitter.im/thingsboard/chat)
 - [Q&A forum](https://groups.google.com/forum/#!forum/thingsboard)
 - [Stackoverflow](http://stackoverflow.com/questions/tagged/thingsboard)

## Licenses

This project is released under [Apache 2.0 License](./LICENSE).
