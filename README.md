
** Work in progress... we are building the documentation, so please be patient!**

# GIoTTY  (Yet Another IoT Plaform)
## Introduction
GIoTTY is Gimasi's IoT platform. The keyword is convention over configuration as well as open source technolgies, we are leveraging the most used and commons frameworks: node.js, RabbitMQ, InfluxDB, Postgres, Cassandra and more to come as we grow.<br/>

The platform was born to handle LoRaWAN nodes but has grown with connectors for practically all IoT technologies..<br/>

The core is a pipeline of Exchanges - driven by RabbitMQ - every step is handled by user 'scripts' which will enrich the message as it flows by.<br/>
	
				Log => Decoder => Alerts => Application Scripts => DataServers / DashBoards

The node data enters the pipeline and is logged, then a Decoder script is executed which transforms the raw payload data in schema variables - every node has a predefined schema. The alert script, application alert, is then executed; here the user can specify application alerts that can be fired. The final 'Application Script' is executed to handle complex logic or define where the data
The Application script can handle data trasmission to other node to achieve sensor/actuator logic.



While GIoTTY is a complete standalone IoT plaform, which includes also dashboards and reporting, it can be used as a middleware to connect physical nodes to other IoT platforms/frameworks, we have connectors for Microsoft Azure, IBM Watson and Amazon AWS and more can be added, being everything scriptable by the user. 


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


## RAW MESSAGE

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


## DECODER

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
