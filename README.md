
** Work in progress... we are building the documentation, so please be patient!**

# GIoTTY  (Yet Another IoT Platform)

## Introduction

GIoTTY is Gimasi's IoT platform/framework. The keyword is convention over configuration.<br/>
The ambition of GIoTTY, as of any IoT platform, is to connect physical nodes which can be sensors or actuators, driven by different IoT technologies ( LoRaWAN, ZigBee, NBIoT etc.), acquiring data, compute and store it locally or simply forward it to other application endpoints. It works also the other way and when you need to send back data to actuators either directly or again from other application.<br/>

GIoTTY can be seen as an IoT HAL (Hardware Abstraction Layer) that connects different technologies converting everything to JSON messages that can be handled easily removing the effort of the underlining protocols and RF technologies.<br/>
GIoTTY is a set of technologies which can be leveraged through an <b>API</b> [link to come] or via the GIoTTY Cloud Application [link to come].

The core of GioTTY is based on an message queue which is interleaved by scripts that are executed as node data passes through, enriching its payload with additional information.<br/>

The message queue steps are:<br/>

        [need picture to describe this]
        Uplinks => Decoder => Alerts => Application Scripts => Endpoints 

This message queue runs from the node to the final 'Endpoint'.<br/>

There is a special internal 'Endpoint':
        DashBoards

Which is the GIoTTY fully customized HTML5 Dashboard system, allowing to build  'fully featured' Dashboards for your IoT data.<br/>

As introduced above there is also a queue that runs the 'other' way, from the Application Server to the nodes. This is used to actuate actuators on the field.

      [need picture to describe this]
      Downlinks => Output

<br/>
The power of the whole architecture that all scripts are user defined so a complete vertical IoT application can be delivered just by simply supplying the correct scripts in the pipeline, via its API or the Cloud Application.


## MVC in IoT or DASE
The whole design mimics the MVC pattern delivering an IoT metaphor DA-S-E<br>

* Model: GIoTTY <b>Decoder / Alert</b>
* Controller: GIoTTY <b>Application Script</b>
* View: GIoTTY <b>Endpoint</b>

The Controller can interfere with the final 'rendering' of the IoT 'View', redirecting or adding the final Endpoint where data should arrive.

## Core Pipeline and User Scripts

There is a defined data flow of information. Every step is handled by scripts which enrich the message data as it flows through.<br/>
	
				Log => ( user Decoder SCRIPT ) => Decoded Data => (user Alerts SCRIPT ) => Alerts => (user Appliation SCRIPT) => Application Scripts => ( Core Routing SCRIPT ) => Endpoints / DashBoards

The node data enters the pipeline and is logged, then a Decoder script is executed which transforms the raw payload data in schema variables - every node has a predefined schema.<br/> 

The alert script is where the user will put application alerting logic and where alerting endpoints can be set<br/>. 
The final 'Application Script' is executed to handle complex logic, storing to timeseries, actuacte other nodes and  define 'Endpoint' logic</br>
In other IoT platforms scripting is available but it is usually condensed in just one script per node, we have designed this fragmented setup to achieve a better code optimization and reuse. Smaller scripts that are finalized to just one task. For example you could have 1000's of temperature nodes that share the same 'Temperature Decoder' script, but need different Alerting scripts. Or you can have alerting scripts that check if water temperature is reaching boiling value, and this script will work anywhere a 'boiling water' alarm is needed. 


# Node Schema, Decoder and Encoder Scripts
As already anticipated every node can (should) have a schema defined.<br/>
If you have a temperature sensor, it's schema variable would be logically called 'temperature'.
A schema needs a <b>decoder script</b> that runs in parallel with the schema. The decoder script will receive the raw data from the node and convert it to schema variables, that will be used throughout the system.<br/>
Schema variables can be declared both as <b>OUPUTS</b>, in this 'temperature' example the schema variable will be an OUTPUT, or as <b>INPUTS</b> which will be used by actuators when you need to set a value on the node.<br>
For ouptut schema variable you need a corresponding <b>Encoder Script</b> that will convert physical values that have been computed by your scripts to the raw payload data that the node is expecting to receive.<br>
Here an exmple of a Decoder script:<br>
```javascript

var _data = raw_data;
var _schema = schema;


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
    _schema.temperature.value = temperature;

  update_schema = _params;
  
    
}

 ```

 And here an example of an Encoder script:<br>
```javascript

var message = node_data


log(node_data);

// convert Temperature

temperature = parseInt(message.temperature * 100);

log("Input Temp=>"+temperature);

low_byte = temperature & 0xff;

high_byte = temperature & 0xff00;
high_byte = high_byte >> 8;

log("Temperature=>"+high_byte+":"+low_byte);


// convert Relay status

if (message.relay == 0 )
  relay_status = 0;
else
  relay_status = 1;


log("Relay=>"+relay_status);


payload1 = high_byte.toString(16);
if (payload1.length == 1) 
  payload1 = "0"+payload1

log(payload1);

payload2 = low_byte.toString(16);
if (payload2.length == 1) 
  payload2 = "0"+payload2

log(payload2);

payload3 = relay_status.toString(16);
if (payload3.length == 1) 
  payload3 = "0"+payload3

log(payload3);    
  
node_payload = "02"+payload1+payload2+"01"+payload3

 ```


# Scripting
The behaviour of the pipeline can be customised via Javascript sripts. At every stage the scripting engine will provide INPUT objects and accept OUTPUT objects.<br/>

There is also a basic logging function - log(object) - that can be used for very simple debugging functionality. Logs for every script can be viewed direcly from the cloud interface, in the log files you will also find traces of errors.<br>
Here, divided by script type, the various INPUT and OUTPUT objects available and some quick examples.

## Decoder
The script will receive in input the <b>raw_data</b> and the <b>schema</b> objects. The code will then update the schema values and pass them over via the output <b>update_schema</b> object.<br/>
The node schema can be declared via the corresponding API or the GIoTTY cloud application.<br>
The decoder script engine will automatically store schema variables, that have been flagged accordingly, to a <b>Time Series</b> ( see TIME SERIES section ).<br>
If a node is part of a Group, data will be stored in a 'Group' <b>Time Series</B> allowing different kinds of aggregation ( see GROUP section ).

<b>Input</b>

 * raw_data
 This is the message arriving directly from the node, it's payload will vary depending on the RF technology

 * schema
 Here is an example of a schema, which is a list of variables keys with their characteristics:

```javascript

// JSON rappresentation of the schema object
 
 {
 
  "humidity": {"type": "Numeric", "mode":"output", "value":"76", "measurement_unit": "%", "max_value": "", "min_value": "", "store_in_time_series": "false"},
  "airpressure": {"type": "Numeric", "mode":"output", "value":"76", "measurement_unit": "hPa", "max_value": "", "min_value": "", "store_in_time_series": "false"},
  "cellvoltage": {"type": "Numeric", "mode":"output", "value":"76", "measurement_unit": "V", "max_value": "", "min_value": "", "store_in_time_series": "false"}, 
  "stateofcharge": {"type": "Numeric","mode":"output",  "value":"76","measurement_unit": "%", "max_value": "", "min_value": "", "store_in_time_series": "false"}, 
  "airtemperature": {"type": "Numeric", "mode":"output", "value":"76","measurement_unit": "°C", "max_value": "", "min_value": "", "store_in_time_series": "false"}
 
 }


 // Javascript object

 var node_schema = schema;

 node_schema.humidity.value = 50;
 node_schema.humidity.cellvoltage = 50;

 ```

In the decoder script only the schema <b>output</b> variables are exposed. All attributes, apart from value, are read only and cannot be updated by the script via the update_schema object.
If a variable has not been declared in the schema it will not be updated.


```javascript

 // weight variable does not exist and will not be added/updated
 // no errors will be raised it will be simply ignored

 node_scheam.wright = {}
 node_schema.weight.value = 50;
``` 

 <b>Output</b>

 * update_schema
 Updating this object will update the schema values.

```javascript

 // let's set a random temperature between 10 and 30.

 var node_schema = schema;

 node_schema.temperature.value = 10 + ( Math.random()*20 );

// need to assign the update_schema object for the update to be persistent
update_schema = node_schema; 
```

## Alert
The Alert script is used to generate application level alerts. Usually based on the schema values that have been updated in the Decoder script.<br/>
Alerts can be opened or closed, and can have assigned an 'endpoint' that can be called on an open or close event.
Alerts have unique keys that define them - the variable on which you are testing and the an attribute that usually defines the test you are applying. For example, if we want to test an alert for boiling water temperature, the alert keys will be: 'temperature'-'max'.<br/>


{"core": {"external_clouds": {"external_cloud_undefined": {"alert_id": 12543, "alert_type": "external_cloud_undefined", "alert_message": "ERROR_EXTERNAL_CLOUD_EMAIL"}}}, "application": {"humidity": {"type": "max", "status": "active", "message": "Max humidity exceeded.", "alert_id": 12556, "duration": 4739902, "endpoints": [], "last_value": 174.4, "started_at": "2017-08-15T12:29:08.861Z", "updated_at": "2017-08-15T13:48:08.763Z"}, "temperature": {"type": "max", "status": "active", "message": "Max temperature exceeded.", "alert_id": 12557, "duration": 4739901, "endpoints": [], "last_value": 174.4, "started_at": "2017-08-15T12:29:08.862Z", "updated_at": "2017-08-15T13:48:08.763Z"}}}

Here the code example:

```javascript

 // Alert script to test if water is boiling

 var node_schema = schema;

  alert={}

  if (node_schema.temperature.value >= 100)
  {
    alert.variable_name = 'temperature';
    alert.type ='max';
    alert.status = 'open';
    alert.message ="Water is boiling!";
  }

// need to assign the update_schema object for the update to be persistent
update_alerts = alert;
```

Once an alert is opened it will not trigger anymore unless you close it. If you need different alerts on the same variables you need to specify different 'type' attributes.

```javascript

 // Alert script to test if water is boiling

 var node_schema = schema;

  alerts=[;]

  if (node_schema.temperature.value >= 90)
  {
    alert={}

    alert.variable_name = 'temperature';
    alert.type ='max_low';
    alert.status = 'open';
    alert.message ="Water is nearly boiling!";

    alerts.push(alert);
  }

  if (node_schema.temperature.value >= 100)
  {
    alert={}

    alert.variable_name = 'temperature';
    alert.type ='max';
    alert.status = 'open';
    alert.message ="Water is boiling!";

    alerts.push(alert);
  }

// need to assign the update_schema object for the update to be persistent
update_alerts = alerts;
```

<b>Input</b>

* raw_data
This is the message arriving directly from the node, it's payload will vary depending on the RF technology

* schema
Schema variables that have been updated by the Decoder script. Usually these are the variables used to trigger alerts.

* alerts
Object describing current alert status.

<b>Output</b>

* update_alerts

## Application Script
The application script can be seen as the controller in the MVC model.<br/>
All model data arrive to via the input objects 'schema' and 'alerts' and the <b>VIEW</b> can be controlled via the output objects 'out_params','call_endpoints','send_to_node'

<b>Input</b>

* raw_data

* schema

* alerts

<b>Output</b>

* out_params

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
   "policy":"disable","override","add",
   "endpoints":[
                {"endpoint_type":"http", "endpoint_data":{"options":""]},
                {"endpoint_type":"email", "endpoint_data":{"from":"", "to":"", "subject":"", "body":""}},
                {"endpoint_type":"mqtt", "endpoint_data":{} },
                {"endpoint_type":"giotty", "endpoint_data":{"endpoit_id":"89128981928"} }
              ]
}


* send_to_node

 ```javascript
 [

  {"node_id": "", "message_type":"payload|schema", "payload":"_HEX_STRING_","schema":{ "temperature":"25.4","relay":"1"}},
  {"node_id": "", "message_type":"payload|schema", "payload":"_HEX_STRING_","schema":{ "temperature":"25.4","relay":"1"}}
]

```

Here some examples of application scripts.<br>

Change the 'Endpoint'

 ```javascript

log("Running...");
call_endpoints =  { "policy":"add","endpoints":[ {"endpoint_type":"giotty","endpoint_data":{"endpoint_id":"24"} } ] };

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


# DOWNLINKS
To send data to a node you can use the REST interface or passing data through the the 'downlinks' exchange.<br/>
Messages must be authenticated via an Auth Token.<br/>

Here an example of the message to be sent:

```javascript
{
  "auth_token":"hjsa78huh23vhsg9",
  "node_id":"bb8eb961-7d98-471a-bb5e-44ecc9c961fb",
  "message_type":"payload|schema",
  "payload":"_HEX_STRING_",
  "schema": { "temperature":"25.4","relais":"1"}
}
```

# GROUPS
In GIoTTY you can define groups of nodes.<br/>
Groups have special aggregation 'schema' functions can be specified on any of the 'nodes' schema variables.<br>
Once defined the data is automatically accumulated and you will have timeseries for the specified 'aggregration schema variables'.<br>
Groups can be used also to assign scripts to nodes without having to add one by one..

# APPS
In GIoTTY you have to specify at least one <b>App</b> to start adding you nodes.<br/>
The Apps concept is like a namespace so that you can easily separata you nodes and actuators in different 'Applications'. 
Every App will have its own: Nodes, Groups and Dashboards
Scripts are instead transversal to all Apps.

# ENDPOINTS
As described before endpoints are the 'physical' endpoints to where the data will be forwarded. You can developed an application with full logic completely in GIoTTY infrastructure with no need to call external endpoints.<bt/>
Currently we are supporting:

* HTTP Rest 
* Email
* MQTT
* Microsoft Azure

# ALERTS
This functionality will show you the Alerts Log of your applications.
You can see which alert are pending.

# LOGS


# MAPS


# DASHBOARDS



