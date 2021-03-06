.. highlight:: sh

.. _dm-hawkbit-mqtt-demo:

============================================
 hawkBit FOTA and MQTT Demonstration System
============================================

.. todo:: add binary install instructions for all boards
.. todo:: add autogenerated flash layout for all boards

.. warning:: **Technology demonstration system only**.

             While the system described below works as documented, it
             is **unstable**, and its behavior may change incompatibly
             in the future. It is also **not supported**.

Overview
========

This page documents how to set up and use a demonstration system
containing IoT devices and an IoT gateway, which can publish sensor
data from devices to the cloud and perform firmware over the air
(FOTA) updates of the device firmware.

A block diagram of this system is shown here. One or more IoT devices
can connect to the network through the same gateway.

.. figure:: /_static/dm-hawkbit-mqtt/hawkbit-system-diagram.svg
   :alt: Device Management with hawkBit System Diagram
   :align: center
   :figwidth: 5in

Using this demonstration system, you can:

- See live temperature readings from your devices appear in the web
  console provided by a cloud MQTT broker, CloudMQTT.

- Upload a cryptographically signed firmware image to a device
  management server, hawkBit.

- Use hawkBit to install a firmware image onto an IoT device via over
  the air update. The device will boot the update after checking its
  cryptographic signature.

.. _96Boards Nitrogen:
   http://www.96boards.org/product/nitrogen/

.. _96Boards HiKey:
   http://www.96boards.org/product/hikey/

.. _UART Serial Mezzanine:
   https://www.96boards.org/product/uartserial/

Get the Hardware
================

To set up this system, you will need a Linux or macOS workstation
computer, one or more IoT devices, and an IoT gateway.

We currently recommend:

- `96Boards Nitrogen`_ as an IoT device
- `96Boards HiKey`_ as an IoT gateway, with `UART Serial
  Mezzanine`_ for console access

Source for other boards is provided on a best-effort basis.

Prepare the System
==================

.. _building and running hawkBit:
   https://eclipse.org/hawkbit/documentation/guide/runhawkbit.html

.. _hawkBit security:
   https://eclipse.org/hawkbit/documentation/security/security.html

.. _Docker:
   https://www.docker.com/

.. _CloudMQTT:
   https://www.cloudmqtt.com

.. _CloudMQTT Control Panel:
   https://customer.cloudmqtt.com/customer/

.. _Ansible:
   https://www.ansible.com

.. _GitHub guide to SSH keys:
   https://help.github.com/articles/connecting-to-github-with-ssh/

.. _Android platform tools:
   https://developer.android.com/studio/releases/platform-tools.html

This is broken down into the following steps.

- :ref:`dm-hawkbit-mqtt-hawkbit`
- :ref:`dm-hawkbit-mqtt-cloudmqtt`
- :ref:`dm-hawkbit-mqtt-linux`
- :ref:`dm-hawkbit-mqtt-gateway`
- :ref:`dm-hawkbit-mqtt-zephyr`
- :ref:`dm-hawkbit-mqtt-device`

.. _dm-hawkbit-mqtt-hawkbit:

1. Set up hawkBit
-----------------

**Required Equipment**: workstation which supports `Docker`_.

Run a demonstration-grade hawkBit server::

    docker run -dit --name hawkbit -p 8080:8080 linarotechnologies/hawkbit-update-server

.. warning::

   This hawkBit container contains an official
   ``hawkbit-update-server`` artifact build from Maven; however, it is
   for **demonstration purposes only**, and should not be deployed in
   production as-is.

   Among other potential issues, the server has an insecure default
   administrative username/password pair. For more information, see
   the official documentation on `building and running hawkBit`_ and
   `hawkBit security`_.

This container can take approximately 40 seconds for the application
to start for the first time.

After running the hawkBit container, visit http://localhost:8080/UI to
load the administrative interface, and log in with the default
username and password (admin/admin).

Your browser window should look like this:

.. figure:: /_static/dm-hawkbit-mqtt/hawkbit-initial.png
   :align: center
   :alt: hawkBit Administrator Interface

.. note::

   For convenience, you may want to adjust the "Polling Time" in the
   "System Config" area. This will instruct your IoT devices to check
   for updates more frequently. The default is 5 minutes; the minimum
   value is 30 seconds.

Your hawkBit container is now ready for use.

.. _dm-hawkbit-mqtt-cloudmqtt:

2. Set up CloudMQTT
-------------------

**Required Equipment**: workstation computer.

First, create a `CloudMQTT`_ account. The free CloudMQTT plan is
enough to run this demo.

After logging in to your account, go to your `CloudMQTT Control Panel`_,
and create a new instance. Then click on the "Details" button
next to the new instance in your control panel. Record the following
information about the instance:

- CLOUDMQTT_SERVER: the URL of the server
- CLOUDMQTT_PORT: the port to connect to on the server
- CLOUDMQTT_USER: the auto-generated username
- CLOUDMQTT_PASSWORD: the auto-generated password

.. _dm-hawkbit-mqtt-linux:

3. Install the Linux microPlatform
----------------------------------

**Required Equipment**: IoT gateway and workstation to flash the board.

Follow the Linux microPlatform :ref:`linux-getting-started` to set up
a `96Boards HiKey`_ gateway for container-based application
deployment.

If you don't have a HiKey, the Getting Started guide contains
information for other boards, provided on a best-effort basis.

.. _dm-hawkbit-mqtt-gateway:

4. Set Up the IoT Gateway
-------------------------

**Required Equipment**: IoT gateway and workstation to run `Ansible`_.

You'll now use Ansible to set up your IoT gateway to act as a network
proxy for your IoT device to publish sensor data to CloudMQTT, and
fetch updates from hawkBit.

.. include:: iot-gateway-setup-common.include

- From the ``gateway-ansible`` repository, deploy the gateway
  containers using the gateway's IP address and CloudMQTT information
  you recorded earlier::

    ansible-playbook -e "mqttuser=CLOUDMQTT_USER mqttpass=CLOUDMQTT_PASSWORD \
                         mqtthost=CLOUDMQTT_SERVER mqttport=CLOUDMQTT_PORT \
                         gitci=WORKSTATION_IP_ADDRESS tag=latest" \
                     -i GATEWAY_IP_ADDRESS, -u linaro iot-gateway.yml \
                     --tags cloud

  WORKSTATION_IP_ADDRESS in the above command line is the IP address
  of the system which is running the hawkBit server you set up
  earlier. **The comma after GATEWAY_IP_ADDRESS is mandatory**.

.. _dm-hawkbit-mqtt-zephyr:

5. Install the Zephyr microPlatform
-----------------------------------

**Required Equipment**: workstation to install the Zephyr microPlatform
development environment, and IoT device to test installation.

Follow the installation steps in the Zephyr microPlatform
:ref:`zephyr-getting-started` guide.

.. _dm-hawkbit-mqtt-device:

6. Set Up the IoT Device(s)
---------------------------

**Required Equipment**: IoT device and workstation to flash the
device.

If you're using `96Boards Nitrogen`_, build and flash the
demonstration application::

  ./genesis build -b 96b_nitrogen zephyr-fota-samples/dm-hawkbit-mqtt
  ./genesis flash -b 96b_nitrogen zephyr-fota-samples/dm-hawkbit-mqtt

.. include:: pyocd.include

If you don't have a Nitrogen, information for other boards is provided
on a best-effort basis in :ref:`dm-hawkbit-mqtt-devices`.

Use the System
==============

Now that your system is fully set up, it's time to check that sensor
data are being sent to the cloud, and do a FOTA update.

Cloud Sensor Updates
--------------------

From your `CloudMQTT Control Panel`_, load your instance's Details
page and click the "Websocket UI" button to get a live view of data
being sent to the server. You should see new data appear every few
seconds; it will look like this:

.. figure:: /_static/dm-hawkbit-mqtt/cloudmqtt-websocket-ui.png
   :align: center

   MQTT messages from 96Boards Nitrogen appearing in CloudMQTT
   Websocket UI.

You can now connect other subscribers to this CloudMQTT instance,
which can act on the data.

FOTA Updates
------------

Now let's perform a FOTA update. In the hawkBit server UI, you should
see the 96Boards device show up in the "Targets" pane. It will look
like this:

.. figure:: /_static/dm-hawkbit-mqtt/96b-target.png
   :align: center

   96Boards Nitrogen registered with hawkBit.

It's time to upload a firmware binary to the server, and update it
using this UI. To make uploading the binaries to the demonstration
hawkBit server easier, download this Python script to your Zephyr
microPlatform installation directory:

https://raw.githubusercontent.com/linaro-technologies/hawkbit/master/hawkbit.py

.. todo:: hawkbit.py should be a versioned part of the release

From the Zephyr microPlatform installation directory::

    python /path/to/hawkbit.py \
                      -ds 'http://localhost:8080/rest/v1/distributionsets' \
                      -sm 'http://localhost:8080/rest/v1/softwaremodules' \
                      -d 'Nitrogen End-to-end IoT system' \
                      -f outdir/zephyr-fota-samples/dm-hawkbit-mqtt/96b_nitrogen/app/dm-hawkbit-mqtt-96b_nitrogen-signed.bin \
                      -sv "1.0" -p "Linaro" -n "Nitrogen E2E preview" -t os

Above, 1.0 is an arbitrary version number.

You will see an update in the hawkBit UI for the new image:

.. figure:: /_static/dm-hawkbit-mqtt/distribution.png
   :align: center

   Distribution Set representing a signed firmware binary.

You'll now update the device. Before doing so, you can connect to its
serial console via USB at 115200 baud to see log messages during the
upgrade (which should be at ``/dev/ttyACM0`` or so on Linux systems).

- Click on the distribution you uploaded, and drag it over the line in
  "Targets" for your IoT Device.

- You'll next need to confirm the action. Click a button towards the
  bottom of your screen labeled "You Have Actions". This should now
  have a "1" at its top right, since you've assigned the distribution
  to your Nitrogen:

  .. figure:: /_static/dm-hawkbit-mqtt/you-have-actions.png
     :align: center

     Click this button.

- A screen will appear. Select "Save Assign" on this screen:

  .. figure:: /_static/dm-hawkbit-mqtt/action-details.png
     :align: center

     Choose "Save Assign".

Your IoT devices will poll the hawkBit server periodically and will
fetch the update the next time they poll.

.. note::

   By default, devices wait five minutes between polls. If you don't
   want to wait and are using 96Boards Nitrogen, you can press the
   "RST" button on the board to reset it; it will check for updates
   shortly after booting.

While hawkBit is waiting for the device to download and install the
update, a yellow circle will appear next to it in the targets list:

.. figure:: /_static/dm-hawkbit-mqtt/96b-nitrogen-waiting.png
   :align: center

   Waiting for 96Boards Nitrogen to update.

When the device informs hawkBit that the download has been
successfully installed, this will turn into a green check box:

.. figure:: /_static/dm-hawkbit-mqtt/96b-nitrogen-ok.png
   :align: center

   96Boards Nitrogen successfully updated.

If you're connected to the device's serial console, look for output
like this while the update is being downloaded and installed::

    [0031200] [fota/hawkbit] [INF] hawkbit_report_update_status: Reporting action ID feedback: success
    [0031210] [fota/hawkbit] [DBG] hawkbit_query:

    POST /DEFAULT/controller/v1/96b_nitrogen-4c1906d0/deploymentBase/1/feedback HTTP/1.1
    Host: gitci.com:8080
    Content-Type: application/json
    Content-Length: 78
    Connection: close

    {"id":"1","status":{"result":{"finished":"success"},"execution":"proceeding"}}
    [0031570] [fota/tcp] [DBG] tcp_received_cb: FIN received, closing network context
    [0031580] [fota/hawkbit] [DBG] hawkbit_query: Hawkbit query completed
    [0031690] [fota/hawkbit] [INF] hawkbit_install_update: Starting the download and flash process
    [0032990] [fota/hawkbit] [DBG] hawkbit_download_cb: 1%
    [0033740] [fota/hawkbit] [DBG] hawkbit_download_cb: 2%
    [0034440] [fota/hawkbit] [DBG] hawkbit_download_cb: 3%
    [0035290] [fota/hawkbit] [DBG] hawkbit_download_cb: 4%
    [...]
    [0627620] [fota/hawkbit] [DBG] hawkbit_download_cb: 98%
    [0628470] [fota/hawkbit] [DBG] hawkbit_download_cb: 99%
    [0629060] [fota/hawkbit] [INF] hawkbit_install_update: Download: downloaded bytes 212992
    [0629070] [fota/hawkbit] [INF] hawkbit_ddi_poll: Triggering OTA update.
    [0629180] [fota/hawkbit] [INF] hawkbit_ddi_poll: Image id 4 flashed successfuly, rebooting now
    [MCUBOOT] [INF] main: Starting bootloader
    [MCUBOOT] [INF] boot_status_source: Image 0: magic=good, copy_done=0xff, image_ok=0xff
    [MCUBOOT] [INF] boot_status_source: Scratch: magic=unset, copy_done=0x23, image_ok=0xff
    [MCUBOOT] [INF] boot_status_source: Boot source: slot 0
    [MCUBOOT] [INF] boot_swap_type: Swap type: none
    [MCUBOOT] [INF] main: Bootloader chainload address offset: 0x8000
    [MCUBOOT] [WRN] zephyr_flash_area_warn_on_open: area 1 has 1 users
    [MCUBOOT] [INF] main: Jumping to the first image slot
    ***** BOOTING ZEPHYR OS v1.8.99 - BUILD: Aug  3 2017 13:28:24 *****
    [0000000] [fota/main] [INF] main: Linaro FOTA example application

    [Startup output omitted]

    {"id":"1","status":{"result":{"finished":"success"},"execution":"closed"}}

Congratulations! You've just done your first FOTA update using this
system.

Known Issues
============

Issues and observations are logged within Linaro's `Bugzilla issue
tracker
<https://bugs.linaro.org/buglist.cgi?component=IoT%20end-to-end&list_id=12808&product=Linaro%20Technologies>`_.

.. rubric:: Footnotes

.. [#hikeyethernet]

   You can also use a USB Ethernet dongle.
