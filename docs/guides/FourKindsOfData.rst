.. _four-kinds-data:

Four kinds of data
##################

.. contents::
   :local:
   :depth: 2

This section provides a general introduction to the device communication with the cloud in the nRF Asset Tracker project.
This is also available as a stand-alone `blog post on DevZone <https://devzone.nordicsemi.com/nordic/nordic-blog/b/blog/posts/the-four-kinds-of-data-you-need-to-consider-when-developing-an-iot-product>`_.

.. figure:: ./images/data-protocols.jpg
   :alt: Data protocols

   Data protocols

Following are the four types of data used in the communication between the device and the cloud:

* Device state
* Device configuration
* Past state
* Firmware upgrades (FOTA) 

Device state
************

The firmware needs to communicate with the cloud to send position updates and important information about the health of the device.
For example, the battery level is a critical health indicator.
This data is considered as the device state.
Since the latest state of the device is always required, a digital twin is used to store this state on the cloud side.
Whenever the device sends an update, the digital twin is updated.
This allows the web application to access the most recent state of the device immediately without waiting for the device to connect and publish its state.

An important criterion for the robustness of any IoT product is the graceful handling of situations where the device is not connected to the internet.
It is not feasible for a device to be connected all the time since wireless communication is relatively expensive, consumes a lot of energy thereby increasing the power consumption in the device.

The firmware is optimized for ultra-low power consumption, and the design aims in reducing the modem uptime and keeping the modem in the off state as long as possible.
A device made smart decides on sending the data based on the situation.

In passive mode, data is sent when movement is detected.
The accelerometer wakes up the application thread, which then tries to acquire a GNSS fix and a cellular connection.
If movement is no longer detected, the modem is turned off and the application returns to the sleep state.
The passive mode is designed to conserve energy as much as possible.
Nevertheless, it is preferred that the device sends an update occasionally, so that the battery condition is known and conditions such as non-functioning of motion sensor can be detected.

Device configuration
********************

Optimizing the device behavior takes time and while the devices are in the field, sending firmware upgrades for every change is expensive.
The usual firmware sizes are around 500 KB, which is expensive even after compression since it takes some time for a device to download and apply the upgrade.
Additionally, there are costs for transferring the firmware upgrade over the cellular network.
Especially in NB-IoT-only deployments, the data rate is low.
Upgrading a fleet of devices with a new firmware involves orchestrating the roll-out and monitoring the faults.
All these challenges mandate the ability to configure the device, which allows to tweak the behavior of the device until the inflection point is reached with respect to battery life and data granularity.

An interesting configuration option is the sensitivity of the motion sensor.
An action that is considered as a movement varies from one tracked subject to another.
A device has various timeout settings such as the waiting time to acquire a GNSS fix, or the waiting time between updates that are sent when the device is in motion.
The timeout settings have a significant influence on power and data consumption.

When the device is set in an active mode, it sends updates based on a configurable interval regardless of whether motion is detected or not.
This is useful when actively developing the firmware with individual devices or when debugging the device behavior in specific areas and situations.

Device configuration is needed if the device must control a particular parameter.
An example is a smart lock, which needs to manipulate the state of a physical lock.
The backend requires a way to convey the desired state of the lock to the device.
This setting needs to be persistently stored on the cloud side because the device could lose power, crash, or lose the information that decides whether the lock must be open or closed.

In this case, the digital twin is used to immediately store the latest desired configuration of the device on the cloud side.
Hence, the application does not need to wait for the device to be connected to record the configuration change.
The implementation of the digital twin sends only the latest required changes to the device, thus minimizing the amount of data that needs to be transferred to the device.
All changes that follow the last request from the device for its configuration are combined into a single change.

.. _firmware-protocol-timestamping:

Time-stamping
=============

Device state and configuration are timeless data that are always required.
The device sends a GNSS position over the cellular connection and the digital twin is updated.
Thus, the current location of the device is known.
When the device configuration is changed (``A -> A'``) the device will eventually apply the new configuration, and if another configuration change was made while the device was not connected (``A' -> A''``) the device can directly transition to ``A''``.
To make state and configuration changes available over time, all changes can be stored on the cloud side with associated timestamps and made available for retrieval in a time-series manner.

This approach has an inherent problem.
If the battery level measured by the device must be stored with the time it was received by the cloud, the timestamp will not be accurate.
If it took a while for the device to establish the cellular connection to send the update, there is a delay of few minutes between the sampling of the battery voltage and the time when the update is finally delivered to the receiving end.
While this might be acceptable with a sensor that has low volatility, it might not be acceptable in scenarios where it is important to know the exact timestamp related to an event.
Consider a case of tracking parcels where it is important to track if a parcel is dropped.
A few minutes of difference can impact the estimation of the exact state of the parcel (for example, parcel being moved by a person or in transit).

It is important to have precise time measurement on the device.
This can be achieved by combining the following time sources:

* Relative device timestamp (a relative time with microsecond resolution that counts upwards from zero after the device is powered on).
* Cellular network time.
* Time from the GNSS sensor.

.. figure:: ./images/timestamping.jpg
   :alt: Time-stamping

   Time-stamping

Whenever a sensor is read, the value is recorded with the device timestamp.
Once these recorded measurements are ready to be sent (in the presence of a cellular connection and the network time is known), the relative timestamps can be converted to absolute timestamps using the relative timestamps of the network or the GNSS time.

In this way, all data is sent with precise timestamps to the cloud where the device time is used when visualizing the data to accurately reflect the creation time of the datum.

Past state
**********

There can be scenarios when the position updates are collected only when a cellular connection can be established.
Consider a reindeer tracker, which tracks the position of a herd.
The reindeer tracker reports movement only along ridges, but never in valleys.
This is because the cellular signal does not have coverage in remote valleys.
However, the GNSS signal is received from the tracker since the satellites, which are high on the horizon, can send the signal down into the valley.

There are many scenarios where the cellular connection might not be available or might be unreliable, but the reading sensors work.
Robust ultra-mobile IoT products must incorporate such conditions into the normal mode of operation.
The absence of a cellular connection must be treated as a temporary condition, which will eventually resolve and until then normal mode of operation must continue.
This means that the devices must continue to measure and store these measurements in a ring buffer or employ other strategies to decide on the data to be discarded once the memory limit is reached.

Once the device can establish a connection successfully, it will publish the past data in batches (after publishing its most recent measurements).

This is also applicable for devices that control a system.
Such devices must have built-in decision rules and they must not depend on the cloud backend to provide the action to be executed based on the current condition.

Firmware upgrades (FOTA)
************************

Firmware upgrade over the air (FOTA) can be considered as a device configuration.
However, the size of a typical firmware image (500 KB) is two to three times larger than the size of a control message.
Therefore, it can be beneficial to treat it differently.
Typically, an upgrade is initiated by a configuration change.
Once the device acknowledges, the firmware download is initiated.
To reduce the overhead, the firmware download is done out of band using HTTP or HTTPS instead of MQTT.

The firmware upgrades are large compared to other messages.
Hence, to conserve resources, the device might suspend all other operations until the firmware upgrade has been applied.

Summary
*******

The nRF Asset Tracker aims to provide robust reference implementations for the four types of device data.
Even though the concrete implementation differs for each cloud provider, the general building blocks (state, configuration, batched past state, firmware upgrades) are the same.

+-------------------------------------+-------------------------+------------------+-----------+-------------------+
| Cloud                               | State                   | Configuration    | Past data | FOTA              |
+=====================================+=========================+==================+===========+===================+
| :abbr:`AWS (Amazon Web Services)`   | `Device shadow`_        | `Device shadow`_ | MQTT      | `Jobs`_ and HTTPS |
|                                     |                         |                  |           |                   |
|                                     | ``reported``            | ``desired``      |           |                   |
+-------------------------------------+-------------------------+------------------+-----------+-------------------+
| :abbr:`GCP (Google Cloud Platform)` | `Device configuration`_ | `Device state`_  | MQTT      |                   |
+-------------------------------------+-------------------------+------------------+-----------+-------------------+
| :abbr:`Azure (Microsoft Azure)`     | `Device twins`_         | `Device twins`_  | MQTT      | `MQTT and HTTPS`_ |
|                                     |                         |                  |           |                   |
|                                     | ``reported``            | ``desired``      |           |                   |
+-------------------------------------+-------------------------+------------------+-----------+-------------------+

.. _Device shadow: https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html
.. _Jobs: https://docs.aws.amazon.com/iot/latest/developerguide/iot-jobs.html
.. _Device configuration: https://cloud.google.com/iot/docs/concepts/devices#device_configuration>
.. _Device state: https://cloud.google.com/iot/docs/concepts/devices#device_state
.. _Device twins: https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins
.. _MQTT and HTTPS: https://docs.microsoft.com/en-us/azure/iot-hub/tutorial-firmware-update