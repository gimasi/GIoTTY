
** Work in progress... we are building the documentation, so please be patient!**

# GIoTTY  (Yet Another IoT Platform)
## Introduction
GIoTTY is Gimasi's IoT platform. The keyword is convention over configuration.<br/>
The whole goal of GIoTTY, as of any IoT platform, is to connect physical nodes which can be sensors or actuators, driven by different IoT technologies ( LoRaWAN, ZigBee, NBIoT etc.), acquiring data for local computation or simply forwarding it to other application servers, and/or sending back data to actuators either directly or again from other application servers.<br/>
GIoTTY can be seen as an IoT HAL - Hardware Abstraction Layer - that connects different technologies converting everything to...<br/>

The core of GioTTY is based on an message queue which is interleaved by scripts that are executed as the the node data passes through the queue enriching its payload with additional application data.<br/>

The message queue steps are:<br/>

        <b>Uplinks</b> => <b>Decoder</b> => <b>Alerts</b> => <b>Application Scripts</b> => <b>Endpoints</b> 

This message queue runs from the node (on the field) to the final Application Server Endpoint.<br/>

There is a special internal 'Endpoint':
        DashBoards

Which is the GIoTTY fully customized HTML5 Dashboard system, that enables very quickly to build 'fully featured' Dashboard to visualize your IoT data.<br/>

There is also a queue that runs the 'other' way, from the Application Server to the nodes. This is used to actuate actuators on the field.

      <b>Downlinks</b> => <b>Output</b>

## Core Pipeline

The core is a pipeline of Exchanges - driven by RabbitMQ - every step is handled by user 'scripts' which will enrich the message as it flows by.<br/>
	
				Log => Decoder => Alerts => Application Scripts => DataServers / DashBoards

The node data enters the pipeline and is logged, then a Decoder script is executed which transforms the raw payload data in schema variables - every node has a predefined schema. The alert script, application alert, is then executed; here the user can specify application alerts that can be fired. The final 'Application Script' is executed to handle complex logic or define where the data
The Application script can handle data trasmission to other node to achieve sensor/actuator logic.



## Scripting
Users can customize the behaviour of the pipeline via Javascript sripts. At every stage the scripting engine will provide INPUT objects to the script:<br/>

 * node_data 
  Available at all levels which represents that raw node data.

 * decoded_data 
 Available at alert and application script level represents the 'decoded' node data. For example you if you have a temperature sensor, you can declare a temperature variable in the node schema. This variable will have its value ( in this example the temperature ) available in the value attribute, for example:
 ```javascript

	current_temperature = decoded_data.temperature.value

    if ( current_temperature > 100) {
    	// water is boiling...
    }

 ```

We also have <b>output</b> objects, with which the user scripts can comunicate with the core infrastructure of GIoTTY, depending at which level your script is running you can have different objects.
* 
*

### Decoder
### Alert
### Application Script
* out_params
This data is serialized and passed over the 'scripts' key in the queue message


* save_timeseries


 ```javascript
 [
  {"time_series1": {key1:value1, key1:value2 },
  {"time_series2": {key1:value1, key1:value2 }
 ]
 ```

* call_endpoints 


 ```javascript
{  
   "policy":"disable|overrride|add",
   "endpoints":[  
      {  
         "http":{  
            "options":""
         }
      },
      {  
         "email":{  
            "from":"",
            "to":"",
            "subject":"",
            "body":""
         }
      },
      {  
         "mqtt":{  

         }
      },
      {  
         "giotty":"id_of_external_cloud"
      }
   ]
}

 ```

## RAW MESSAGE

```javascript
{ 
  id: 'ed391069-8363-460e-a1b1-8db95cf32434',
  
  instance_id: 'fcc60389-e8ba-4657-b6e1-cd6894171b48',
  company_id: '7ec6394d-9a92-4a28-86e6-eb141e9d722e',
  user_id: 'e23ec145-6cc0-4af7-bdae-6c9c04f3b03b',
  app_id: '9bda6968-3848-4738-8266-76b94521f5d5',

  node_id: 'bb8eb961-7d98-471a-bb5e-44ecc9c961fb',
  
  log_type: 'data_up',
  
  app_id: '9bda6968-3848-4738-8266-76b94521f5d5',
  node_id: 'bb8eb961-7d98-471a-bb5e-44ecc9c961fb',
  node_type_code: 'lora868',
  
  node_multicast: false,
  
  station_id: '831e34f2-55cc-4d89-b2df-2291d60125b4',
  
  gateway_id: '0000AA555A00ABCD',
  
  payload: '010b8f',
  
  snr: 5.1,
  rssi: -35,
  
  other:
    { DataUp:
  
      { MHDR: [Object],
        DevAddr: '48100012',
        Fctrl: [Object],
        FCnt: 0,
        FOpts: '',
        FPort: 4,
        FPayloadType: 'data',
        FPayload: '07f168',
        FPayloadDeciphered: '010b8f',
        FullFrame: '40120010480000000407f168fbbc2dc8',
        MIC: '00000000',
        MIC_validity: true },
     rxpk:
      { time: '2017-08-12T11:08:13.803Z',
        tmst: 1502536093,
        chan: 2,
        rfch: 0,
        freq: 868.5,
        stat: 1,
        modu: 'LORA',
        datr: 'SF10BW125',
        codr: '4/6',
        rssi: -35,
        lsnr: 5.1,
        size: 16,
        data: 'QBIAEEgAAAAEB/Fo+7wtyA==' },
     deveui: '78af580312121215',
     devaddr: '48100012' },
  
  response: null,

  created_at: '2017-08-12T11:08:14.391Z',
  updated_at: '2017-08-12T11:08:14.391Z',
  
  app: { id: '9bda6968-3848-4738-8266-76b94521f5d5' },
  node:
   { id: 'bb8eb961-7d98-471a-bb5e-44ecc9c961fb',
     serial: '78af580312121215',
     node_groups: [ [Object] ] },

  core_session_id: 'e14ad78e-2e06-4fbd-93db-b09375c04dba' 

  }

```


## DECODER
Here is an example script<br/>

```javascript

var _data = node_data;
var _params = in_params;


var payload = _data.payload;

log(_data);

function hexToBytes(hex) {
    for (var bytes = [], c = 0; c < hex.length; c += 2)
        bytes.push(parseInt(hex.substr(c, 2), 16));
    return bytes;
}

var obj ={};

log("Starting");
byte_data = hexToBytes( payload );
log( byte_data );

if ((byte_data[0] == 0x01 ) || (byte_data[0] == 0x02 )){
	var temperature;

	// read sensor temperature
	temperature = byte_data[1] << 8;
	temperature = temperature + byte_data[2];
					
    temperature = temperature / 100;
  
  	log("Temperature=");
    log(temperature);
  	_params.temperature.value = temperature;

	out_params = _params;
   	
}
```

##DOWNLINKS
To send data to a node you can use the REST interface or passing data through the the 'downlinks' RabbitMQ exchange.<br/>
Messages must be authenticated via an Auth Token.<br/>

Here an example of the message to be sent:

```javascript
{
  "auth_token":"hjsa78huh23vhsg9",
  "node_id":"bb8eb961-7d98-471a-bb5e-44ecc9c961fb",
  "message_type":"payload|schema",
  "payload":"_HEX_STRING_",
  "schema":"{ "temperature":"25.4","relais":"1"}
}
```
