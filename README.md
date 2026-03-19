# GEOVIBE REFACTORING

# new imports
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
// No imports are needed for Thread (java.lang by default)

# CHANGES TO THE MqttClientManager

1) to method connect(MqttCallback callback) added follow:
   MqttConnectOptions connOpts = new MqttConnectOptions();
   connOpts.setKeepAliveInterval(3600); // 1 hour
   connOpts.setConnectionTimeout(180);  // 3 minutes
   connOpts.setAutomaticReconnect(true);  // auto-reconnect
   connOpts.setCleanSession(false);     // saving the session
   connOpts.setUserName(config.getUserName());
   connOpts.setPassword(config.getPassword().toCharArray());


2) ensureIsConnected(MqttCallback callback) method refactored
The reconnectAttempts counter now serves purely informative purposes (you can see in the log that this is the 120th attempt in 2 hours).
Resetting to 0 on success is correct.
Thread.sleep(RECONNECT_DELAY_MS) is acceptable here, since now this method calls the background thread, not the main server thread.

# CHANGES TO THE MqttTransportBase

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
How it works: The thread "sleeps" for 60 seconds, wakes up, checks the status. If there is no connection, ensureIsConnected wakes up. If you click "Stop" in GeoEvent, interrupt() will be called, the stream will exit sleep and end.

4) remove setErrorState() from start() or run()

5) method stop()
isMonitoring = false: This is your "soft" lever. The thread will check the while (isMonitoring) condition and exit the loop itself.
monitorThread.interrupt(): This is an "alarm clock". If the thread is sleeping at this moment (Thread.sleep(60000)), it will not wait for the rest of the minute, but will immediately wake up, catch an InterruptedException and terminate.
monitorThread.join(2000): We give the stream time for a "clean" closure. This prevents the situation when the old thread tries to reach the already closed MQTT client.
synchronized: Ensures that if someone presses the "Stop" button twice very quickly, the system will not try to stop the same thread twice at the same time.


# Using logic from IotHubMqttMessageBase

The run() code from IotHubMqttMessageBase can be adapted for status monitoring.:

Run a separate thread that checks MqttClient.isConnected() every 5-10 minutes.
When disconnected, run ensureIsConnected().
If disconnected for a long time (>1 hour), log a warning, but do not stop the service.

# Where do the methods come from (Paho vs Java)

 setAutomaticReconnect, setCleanSession, and setKeepAliveInterval are methods of the MqttConnectOptions class from the Eclipse Paho library (org.eclipse.paho.client.mqttv3). This is an implementation of the MQTT protocol in Java. If the project is compiled, it means that the imports are in place.

 Thread.interrupt() and setDaemon() are pure Core Java (standard Thread class). You don't need any implants.

# weak spots
1) KeepAlive in MQTT is not about the "collapse of traffic" (data), but about the "channel status". If there is no data for 10 hours, but the TCP connection is alive, MQTT will send tiny PING packets.
The risk of a long interval: If the network actually drops (physically), your client will consider himself "connected" for another hour. At this time, GeoEvent may try to send data "to nowhere", and it will accumulate or get lost without giving an error.

2) But while our main task is to prevent GeoEvent from "spitting out" the connector (converting it to Error) and closing it, then this implementation is suitable.

    Why this will work: Your monitorThread in MqttTransportBase will spin endlessly while(isMonitoring). Even if the Internet goes out for a day, the stream will try to call ensureIsConnected once a minute. As soon as the network appears, the connection will be restored by itself.

    The 10 o'clock nuance: In order for the messages that have accumulated on the broker to reach GeoEvent after 10 hours of downtime, be sure to use connOpts.setCleanSession(false) and make sure that your ClientID in the GeoEvent config is static (does not change upon reboot)

# FINAL AUDIT OF CHANGES
1. Life cycle and "Observation Layer"
Previously, MqttTransportBase relied on the start() method, which triggered the connection once. If the connection was broken, the transport went into ERROR and "died" before a manual restart.
Now the logic is as follows:
 At startup (run): The monitorThread starts. This is an endless loop that lives parallel to the main logic of GeoEvent.
 Every minute: The monitor pulls mqttClientManager.ensureIsConnected().
 Interaction: MqttTransportBase (Observer) constantly checks MqttClientManager (Executor). If the contractor reports !isConnected(), the observer forces it to reconnect.
2. Protection against "Race Condition"
Since we now have two connection command sources (the internal Paho auto-reconnect and your monitorThread), using AtomicBoolean isConnecting is critical.
    Without it: The monitor sees that there is no connection and calls connect(). At the same moment, the Paho library tries to reconnect itself. A conflict occurs, an MqttException is thrown, and garbage is thrown into the logs.
    With him: The first one who captures the isConnecting flag does the work. The second one just peacefully exits the method, seeing that "the process is underway." This makes the system stable and quiet in the logs.

3. The "Survival" Config
The changes to MqttConnectOptions inside the MqttClientManager are the foundation that allows you not to "fall off" in 10 hours.:
<img width="630" height="399" alt="image" src="https://github.com/user-attachments/assets/ddc186f9-482b-46de-989e-b95227d0178c" />

4. Behavior during 10-hour downtime (Scenario)
    Hour 0: The Internet goes out. connectionLost records this in the log.
    Hour 1-9: monitorThread wakes up once a minute, sees !isConnected(), calls ensureIsConnected. He tries to reach the broker, gets an error (there is no network), writes a short WARNING to the log and falls asleep. At the same time, GeoEvent shows the status of the connector "Started" (green), kafka services are not stopped.
    Hour 10: The Internet appears. monitorThread calls connect() in the next iteration. Thanks to Clearance(false), the broker recognizes the old client and dumps all the messages accumulated over 10 hours into GeoEvent.

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
