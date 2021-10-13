# Child-Devices - Measurements

That document specifies the thin-edge child-devices functionality in terms of "Measurements".

## Scope
The thin-edge Interface for measurements (MQTT topic "tedge/measurements") is used to push measurement-data to the thin-edge device's object in the cloud.

Same interface shall be used to push measurement-data to thin-edge device's child-device objects in the cloud. What ever the connected cloud can understand as child-device object is implemented in the cloud specific mapper.

## Interface on local MQTT bus

### Pushing measurements to thin-edge device object
**TOPIC:** ```tedge/measurements <thin-edge JSON payload> ```<br/>
**PAYLOAD:** ```<thin-edge JSON payload>```

Example:
```tedge/measurements { "temp": 10.1 }```

### Pushing measurements to thin-edge's child-device object
**TOPIC:** ```tedge/measurements/<child-device ID>```<br/>
**PAYLOAD:** ```<thin-edge JSON payload>```

Child-Device ID is a string that contains what ever the connected Cloud uses as child-device ID.

Example:
```tedge/measurements/child1 { "temp": 10.1 }```


## Cloud interaction, as by C8Y as example
JSON-via-MQTT message from thin-edge to C8Y.

### Pushing measurements to thin-edge device object
**TOPIC:** ```c8y/measurement/measurements/create``` <br/>
**PAYLOAD** 
```{
   "type":"ThinEdgeMeasurement",
   "temp": {
      "temp":{
         "value":10.1
      }
   },
   "time":"2021-10-12T08:18:06.145201746+01:00"
}
```

### Pushing measurements to thin-edge's child-device object
**TOPIC:** ```c8y/measurement/measurements/create``` <br/>
**PAYLOAD** 
```{
   "type":"ThinEdgeMeasurement",
   "externalSource": {         // reference for c8y identity API
      "externalId":"child1",   // to store measurement in object with 
      "type":"c8y_Serial"      // external ID "child1"
   },
   "temp": {
      "temp":{
         "value":10.1
      }
   },
   "time":"2021-10-12T08:18:06.145201746+01:00"
}
```

# Reference Prototype
More details about concept and implementation needs are demonstrated in prototype of pull request below: 

   https://github.com/cstoidner/thin-edge.io/pull/1 
