GEOVIBE REFACTORING
=======================

CHANGES TO THE MqttClientManager
=======================
1) to method connect(MqttCallback callback) added follow:
   MqttConnectOptions connOpts = new MqttConnectOptions();
   connOpts.setKeepAliveInterval(3600); // 1 hour
   connOpts.setConnectionTimeout(180);  // 3 minutes
   connOpts.setAutomaticReconnect(true);  // auto-reconnect
   connOpts.setCleanSession(false);     // saving the session
   connOpts.setUserName(config.getUserName());
   connOpts.setPassword(config.getPassword().toCharArray());


2) then:
   private int reconnectAttempts = 0;
   private static final int MAX_RECONNECT_ATTEMPTS = 0; // 0 = infinite reconnection
   private static final int RECONNECT_DELAY_MS = 60000; // the delay between attempts is 1 minute

and ensureIsConnected(MqttCallback callback) method refactoring:
    public void ensureIsConnected(MqttCallback callback) throws MqttException {
       if (!isConnected() {  //DELETED if (!isConnected() && (MAX_RECONNECT_ATTEMPTS == 0 || reconnectAttempts < MAX_RECONNECT_ATTEMPTS))
         disconnect();
         try {
           Thread.sleep(RECONNECT_DELAY_MS); // pause before reconnecting
           connect(callback);
           reconnectAttempts = 0; // Resetting the counter on success 
           LOGGER.info("MQTT reconnected successfully.");
           // DELETED reconnectAttempts++;
         } catch (InterruptedException e) {
           LOGGER.error("Interrupted during reconnect attempt", e);
         } catch (MqttException e) {
         reconnectAttempts++;
         LOGGER.warn("Reconnect failed (attempt #{0}). Will retry.", reconnectAttempts, e);
         // продолжаем попытки — фоновый монитор вызовет снова
       }
       }
     }


CHANGES TO THE MqttTransportBase
=======================
1) to the connectionLost(Throwable cause) method, we add the fault accounting logic:
 @Override
  public void connectionLost(Throwable cause)
  {
    LOGGER.debug("CONNECTION_LOST", cause, cause.getLocalizedMessage());
    LOGGER.warn("MQTT connection lost: {}", cause.getMessage());
    LOGGER.warn("MQTT connection lost: {}. Reconnect will be attempted by monitor.", cause.getMessage());
  }

2) Added configuration parameters to MqttTransportBase
   private static final long MONITOR_INTERVAL_MS = 60000; // Проверка каждые 60 секунд
   private volatile boolean isMonitoring = false;
   private Thread monitorThread;


3) added startConnectionMonitor method for BACKGROUND MONITORING and apply in run() and stop() methods

4) remove setErrorState() from start() or run()

=======================
Using logic from IotHubMqttMessageBase

The run() code from IotHubMqttMessageBase can be adapted for status monitoring.:

Run a separate thread that checks MqttClient.isConnected() every 5-10 minutes.
When disconnected, run ensureIsConnected().
If disconnected for a long time (>1 hour), log a warning, but do not stop the service.


=======================
# mqtt-for-geoevent

ArcGIS GeoEvent Server sample MQTT inbound and outbound transports for sending and receiving messages in the MQTT format.

![App](mqtt-for-geoevent.png?raw=true)

## Features
* MQTT Inbound Transport
* MQTT Outbound Transport

## Instructions

Building the source code:

1. Make sure Maven and ArcGIS GeoEvent Server SDK are installed on your machine.
2. Run 'mvn install -Dcontact.address=[YourContactEmailAddress]'

Installing the built jar files:

1. Copy the *.jar files under the 'target' sub-folder(s) into the [ArcGIS-GeoEvent-Server-Install-Directory]/deploy folder.

## Requirements

* ArcGIS GeoEvent Server.
* ArcGIS GeoEvent Server SDK.
* Java JDK 1.8 or greater.
* Maven.

## Resources

* [ArcGIS GeoEvent Gallery](http://links.esri.com/geoevent-gallery)
* [ArcGIS GeoEvent Server Resources](http://links.esri.com/geoevent)
* [ArcGIS Blog](http://blogs.esri.com/esri/arcgis/)
* [twitter@esri](http://twitter.com/esri)

## Issues

Find a bug or want to request a new feature?  Please let us know by submitting an issue.

## Contributing

Esri welcomes contributions from anyone and everyone. Please see our [guidelines for contributing](https://github.com/esri/contributing).

## Licensing
Copyright 2017 Esri

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

A copy of the license is available in the repository's [license.txt](license.txt?raw=true) file.
