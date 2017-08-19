
** Work in progress... we are building the documentation, so please be patient!**

# GIoTTY  (Yet Another IoT Platform)

## Introduction

GIoTTY is Gimasi's IoT platform/framework. The keyword is convention over configuration.<br/>
The ambition of GIoTTY, as of any IoT platform, is to connect physical nodes which can be sensors or actuators, driven by different IoT technologies ( LoRaWAN, ZigBee, NBIoT etc.), acquiring data, compute and store it locally or simply forward it to other application endpoints. It works also the other way and when you need to send back data to actuators either directly or again from other application.<br/>

GIoTTY can be seen as an IoT HAL (Hardware Abstraction Layer) that connects different technologies converting everything to JSON messages that can be handled easily removing the effort of the underlining protocols and RF technologies.<br/>
GIoTTY is a set of technologies which can be leveraged through an <b>API</b> [link to come] or via the GIoTTY Cloud Application [link to come].

The core of GioTTY is based on an message queue which is interleaved by scripts that are executed as node data passes through, enriching the message payload with additional information.<br/>

The message queue steps are:<br/>

        [need picture to describe this]
        Node => Logging => Decoder => Alerts => Application Scripts => Endpoints 

This message queue runs from the node to the final 'Endpoint'.<br/>

There is also a queue that runs the 'other' way, from the an 'Endpoint' to a node/s. This is used to actuate actuators on the field.

      [need picture to describe this]
      Endpoints => Downlinks => Node

<br/>
The power of GIoTTY comes from the user definable scripting architecture. A complete vertical IoT application can be delivered just by simply supplying the correct scripts in the pipeline, either via its API or the Cloud Application.


# Scripting and Core Pipeline

Scripts are categorized based on their execution position on the pipeline. Based on the 'category' or position on the pipeline the scripting engine will provide different INPUT and OUTPUT objects to the script<br/>

Here is a diagram of the core message queue/pipeline, showing where the different type of user scripts are executed.<br/>


				[need picture to describe this]		
				Logging => ( user Decoder SCRIPT ) => Decoded Data => (user Alerts SCRIPT ) => Alerts => (user Appliation SCRIPT) => Application Scripts => ( Core Routing SCRIPT ) => Endpoints 


The node data enters the pipeline and is logged, then a Decoder script is executed which transforms the raw payload data in schema variables - every node has a predefined schema, which defines the properties it measures.<br/> 

The alert script is where the user will put application alerting logic and where alerting endpoints can be set<br/>. 
The final 'Application Script' is executed to handle complex logic, storing to timeseries, actuacte other nodes and  define 'Endpoint' logic</br>
In other IoT platforms scripting is available but it is usually condensed in just one script per node, we have designed this fragmented setup to achieve a better code optimization and reuse. Smaller scripts that are finalized to just one task. For example you could have 1000's of temperature nodes that share the same 'Temperature Decoder' script, but need different Alerting scripts. Or you can have alerting scripts that check if water temperature is reaching boiling value, and this script will work anywhere a 'boiling water' alarm is needed. 


There is also the 'reverse' pipeline has only one script 'slot':

				Enpoints => Downlinks => ( user Encoder Script ) => Nodes

The Enconder script convert the schema variables to the raw payload data that the node is expecting to receive.


## MVC in IoT or DASE
The whole design mimics the MVC pattern delivering an IoT metaphor DA-S-E<br>

* Model: GIoTTY <b>Decoder / Alert</b>
* Controller: GIoTTY <b>Application Script</b>
* View: GIoTTY <b>Endpoint</b>

The Controller can interfere with the final 'rendering' of the IoT 'View', redirecting or adding the final Endpoint where data should arrive.


# Node Schema, Decoder and Encoder Scripts
Every node can (should) have a schema defined.<br/>

If you have a temperature sensor, it's schema variable would be logically called 'temperature', here a JSON example of its definition:<br/>

```javascript

// JSON rappresentation of the schema object for a single variable 'temperature'
 
 {
    "temperature": 
    	{
    	  "mode": "OUTPUT", 
    	  "value": "",
    	  "measurement_unit":"°C",
    	  "min_value": "10", 
    	  "max_value": "50", 
    	  "store_in_time_series": true|false
    	}
 }

``` 

Schema variables can be declared both as <b>OUTPUTS</b>, in this 'temperature' example the schema variable will be an OUTPUT, or as <b>INPUTS</b> which will be used by actuators when you need to set a value on the node.<br>
The <b>value</b>b> attribute will be updated by the decoder script.<br>
The <b>measurement_unit</b>b> attribute is a string value that can be used in UI creation.<br/>
<b>min_value</b> and <b>max_value</b> are currently used only for UI widgets creation, but could be used for alerting generation via the alert scripts.<br/>
<b>store_in_time_series</b> flag will enable automatic Time Series storage.<br/>
<br>
A schema needs a <b>Decoder Script</b> that runs in parallel with the schema itself. The decoder script will receive the raw data from the node and convert it to schema variables, that will be used throughout the pipeline.<br/>
The decoder script will add an additional attribute <b>updated_at</b> that will contain the timestamp of the last update of the schema <b>value</b>.<br/>
For the INPUT schema you need a corresponding <b>Encoder Script</b> that will convert physical values that have been computed by your scripts to the raw payload data that the node is expecting to receive.<br>


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

Here, divided by script type, the various INPUT and OUTPUT objects available and some quick examples.

## Decoder
The script will receive in input the <b>raw_data</b> and the <b>schema</b> objects. The code will then update the schema values and pass them over via the output <b>update_schema</b> object.<br/>
The decoder script engine will automatically store schema variables, that have been flagged accordingly, to a <b>Time Series</b> ( see TIME SERIES) section ).<br>
If a node is part of a Group, data will be stored in a 'Group' <b>Time Series</b> allowing different kinds of aggregation. Also if a Group has a Decoder Script assigned all nodes part of the group will inherit that script ( see GROUP section for full description).

<b>Input</b>

 * <b>raw_data</b><br/>
 This is the message arriving directly from the node, it's payload will vary depending on the RF technology.<br/>

 Here an example of the LoRaWAN raw message payload:<br/>
 
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


 * <b>schema</b><br/>
 The schema available in the object reflects the schema defined at node level with the last updated values, <b>value</b> and <b>updated_at</b>
 Here is an example of a schema, which is a list of variables keys with their attributes:

```javascript
 
 // JSON rappresentation of the schema object for a single variable 'temperature'
 {
   "temperature": 
   		{ 
   			"mode": "OUTPUT", 
   			"measurement_unit":"°C",
   			"value": 28.41, 
   			"max_value": "10", 
   			"min_value": "50", 
   			"updated_at": "2017-08-18T09:23:37Z", 
   			"store_in_time_series": true
   		}
  }
 

 // Javascript object

 var node_schema = schema;

 current_temperature = node_schema.temperature.value ;
 measurement_unit = node_schema.temperature.measurement_unit ;
 
 message = "Temperature is "+current_temperature+ " "+measurement_unit;
 ```

In the decoder script only the schema <b>OUTPUT</b> variables are exposed. All attributes, apart from value, are read only and cannot be updated by the script via the update_schema object.<br/>

 <b>Output</b>

 * <b>update_schema</b><br/>
 Updating this object will update the schema values.

```javascript

 // let's set a random temperature between 10 and 30.

 var node_schema = schema;

 node_schema.temperature.value = 10 + ( Math.random()*20 );

// need to assign the update_schema object for the update to be persistent
update_schema = node_schema; 
```

If a variable has not been declared in the schema it will be ignored..

```javascript

 node_scheam.weight = {}
 node_schema.weight.value = 50;

 // weight variable does not exist and will not be added/updated
 // no errors will be raised it will be simply ignored

 update_schema = node_schema; 

``` 

## Alert
The Alert script is used to generate application level alerts. Usually based on the schema values that have been updated by the Decoder script.<br/>
Alerts can be opened or closed, and can have assigned an 'endpoint' that can be called on an open or close event.
Alerts have unique keys that define them - the variable on which you are testing and the an attribute that usually defines the test you are applying. For example, if we want to test an alert for boiling water temperature, the alert keys will be: 'temperature'-'max'.<br/>
The alerts are set by passing an array of alerts to the <b>update_alerts</b> object. Here is the JSON structure of the alerts array:

```javascript

[
    {"variable_name":"","type":"", "status":"open|close", "value":"", "message":"", "endpoints":[ ]},
    {"variable_name":"","type":"", "status":"open|close", "value":"", "message":"", "endpoints":[ ]}
]

```

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

 // Multi level alert on same variable

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
<br/>
Here some more complex examples with endpoints that are called when the alerts are triggered:

```javascript

var _decoded = schema;
var _alert = alerts;


update_alerts = [];

var temp_alert = (Math.random()*10)+30;

var temperature_alert = {}

if (_decoded.temperature.value > temp_alert) 
{
  
   temperature_alert.message = "Temperature Alert for node:"+raw_data.node.serial+"\n Temperature value is:"+_decoded.temperature.value+" while alert level is:"+temp_alert+"\n";
  
   temperature_alert.variable_name = 'temperature';
   temperature_alert.alert_type = 'max';
   temperature_alert.status = 'open';
   temperature_alert.value = _decoded.temperature.value;
   temperature_alert.endpoints = [  {"endpoint_type":"email", "endpoint_data":{"from":"info@gimasi.ch", "to":"info@gimasi.ch", "subject":"Giotty Alert!", "body": temperature_alert.message} } ]
 } 
else
{
   message = "Temperature Alert for node:"+raw_data.node.serial+" Has been closed! \n";
  
   temperature_alert.variable_name = 'temperature';
   temperature_alert.alert_type = 'max';
   temperature_alert.status = 'close';
   temperature_alert.endpoints = [  {"endpoint_type":"email", "endpoint_data":{"from":"info@gimasi.ch", "to":"info@gimasi.ch", "subject":"Giotty Alert Closed!", "body": message} } ]

}

update_alerts.push(temperature_alert);

```

<b>Input</b>

* <b>raw_data</b><br/>
This is the message arriving directly from the node (same as in the Decoder script).

* <b>schema</b><br/>
Schema variables that have been updated by the Decoder script. Usually these are the variables used to trigger alerts.

* <b>alerts</b><br/>
An array describing the current alerts status. If an alert is continously opened, ie. the alert is still on,  it is updated, with the updated_at and duration variables that reflect the last update time.<br/>

Here the JSON object of alerts

```javascript
[
"temperature": {"type": "max", "status": "active", "message": "Max temperature exceeded: 20.19 > 11.802603092029429", "alert_id": 12689, "duration": 10291, "endpoints": [], "last_value": 20.59, "started_at": "2017-08-16T14:33:45.616Z", "updated_at": "2017-08-16T14:33:55.907Z"}}}
]
```
* <b>storage</b><br/>
The <b>storage</b> objects gives acces to a series of storage function that allow the script to persist data. There are two storages - local and global. The local storage is accessible only to the node while the global storage is accessible to all nodes in the Application. To use the global storage you have 2 methods get and set, that will store a key value pair, instead to use the local storage you need to update the <b>update_local_storage</b> OUTPUT object <b>update_storage</b> ( more information in the STORAGE section )<br/>
Here is an example:<br/>

```javascript

object = {};
object.node_id = raw_data.node_id;
object.variable = 1;

log("****STORAGE***");
log( raw_data.core_session_id);

log("NODE:"+raw_data.node_id);

value = storage.global.get("myVariable");
if (!value)
{
    log("myVariable is not set");  	    
    variable = "The node that set the variable is :"+raw_data.node_id;        
    storage.global.set("myVariable",variable)
}

if (raw_data.node_id == 'bb8eb961-7d98-471a-bb5e-44ecc9c961fb')
  update_local_storage = object;

```

<b>Output</b>

* update_alerts<br/>
As shown in the example above the object that will receive the data for openning/closing alerts.

* update_local_storage<br/>
To store local node persistent data.

## Application Script
The application script can be seen as the controller of the MVC model.<br/>
The INPUT objects ( the Model of the MVC ) are 'raw_data','schema' and 'alerts' and the <b>VIEW</b> behaviour can be controlled via the output objects 'out_params','call_endpoints','send_to_node'.

<b>Input</b>

* <b>raw_data</b><br/>
The data arriving directly from the node (same as in the Decoder and Alert script).

* <b>schema</b><br/>
The Decoded 

* <b>alerts</b>

* <b>storage</b>

<b>Output</b>

* <b>out_params</b>

* <b>save_timeseries</b>

 ```javascript
  [
  {"time_series1": {key1:value1, key1:value2 },
  {"time_series2": {key1:value1, key1:value2 }
 ]
 ```

* <b>call_endpoints</b>

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
```

* <b>send_to_node</b>

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
call_endpoints =  { "policy":"add","endpoints":[ {"endpoint_type":"giotty","endpoint_data":{"endpoint_id":"bb8eb961-7d98-471a-bb5e-44ecc9c961fb"} } ] };

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

# TIME SERIES
To store node data and build historic series you can use the Time Series functionality.<br/>
Time Series can be automatically generated at node level, by flagging the attribute in the node schema, or can be generated by the Application Script.<br/>


# GROUPS
In GIoTTY you can define groups of nodes.<br/>
Groups have special aggregation 'schema' functions can be specified on any of the 'nodes' schema variables.<br>
Once defined the data is automatically accumulated and you will have timeseries for the specified 'aggregration schema variables'.<br>
The aggregation functions available are:<br/>
* COUNT()
* DISTINCT()
* INTEGRAL()
* MEAN()
* MEDIAN()
* MODE()
* SPREAD()
* STDDEV()
* SUM()

Groups can be used also to assign scripts to nodes without having to add one by one, this is particulary

# APPS
In GIoTTY you have to specify at least one <b>App</b> to start adding you nodes.<br/>
The Apps concept is like a namespace so that you can easily separata you nodes and actuators in different 'Applications'. 
Every App will have its own: Nodes, Groups and Dashboards
Scripts are instead transversal to all Apps.

# ENDPOINTS
As described before endpoints are the 'physical' endpoints to where the data will be forwarded. You can developed an application with full logic completely in GIoTTY infrastructure with no need to call external endpoints.<bt/>
Currently we are supporting:

* <b>HTTP</b> <br/>
The HTTP Rest call is implemenent with the [request](https://www.npmjs.com/package/request) npm module. We are exposing the whole options parameters so that you can build anything that is possible with the request module.<br/>
You need to specify the parameters as pure JSON ( see example below ) we have added also [Mustache](https://github.com/janl/mustache.js) templating engine that parses the whole JSON, so you can bind any parameter coming from the core scripting engine ( raw_data, schema, alerts or script ) to achieve fully dynamic calls based on the node message content.
Here a quick example:<br/>

```javascript
{
  "method":"GET",
  "url":"http://www.gimasi.ch",
  "qs":{ "node_id":"{{raw_data.node.id}}","payload":"{{raw_data.payload}}"}
}
```

* <b>Email</b><br/>
This is very simple, you have to fill a from, to, subject and body. In this initial version we are sending plain text emails. Mustache templating engine is available on the subject and body field.<br/>

```javascript
{
  "from":"me@me.com",
  "to":"you@you.com",
  "subject":"Information from node {{raw_data.node.id}}"
  "body":"The temperature in the kitchen is {{schema.temperature.value}} {{schema.temperature.measurement_unit}}"
}

```

* MQTT
* Microsoft Azure

There is a special internal 'Endpoint':
* DashBoards

Which is the GIoTTY fully customized HTML5 Dashboard system, allowing to build  'fully featured' Dashboards for your IoT data.<br/>

# ALERTS
This functionality will show you the Alerts Log of your applications.
You can see which alert are pending.

# LOGS


# MAPS


# DASHBOARDS
