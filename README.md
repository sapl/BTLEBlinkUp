# BTLEBlinkUp 2.0.0 #

This library provides a foundation for activating end-user devices via Bluetooth LE on imp modules that support this wireless technology (currently imp004m and imp006 only) using BlinkUp™.

The BTLEBlinkUp library is intended for Electric Imp customers **only**. It contains device-side Squirrel code which enables BlinkUp device activation using enrollment and WiFi credentials transmitted by a companion app running on a mobile device. The companion app must contain the [Electric Imp BlinkUp SDK](https://developer.electricimp.com/manufacturing/sdkdocs), use of which is authorized by API key. Only Electric Imp customers can be provided with a suitable BlinkUp API key. Code for the companion app is not part of this library, but iOS example code is [available separately](https://github.com/electricimp/BluetoothBlinkUp).

**To include this library in your project, add** `#require "bt_firmware.lib.nut:1.0.0"` **and** `#require "btleblinkup.device.lib.nut:2.0.0"` **at the top of your device code**.

## Example Code ##

Please see the repo [BluetoothBlinkUp](https://github.com/electricimp/BluetoothBlinkUp) for working iOS and Squirrel application code which makes use of this library.

## Class Usage ##

### Application Code Loading Time ###

In order to support BlinkUp for device activation, the production application code which makes use of this library must be running when the device is first powered up by an end-user. To ensure this happens, you **must** have your [Connected Factory](https://developer.electricimp.com/manufacturing/factoryprocess) configured so that newly assembled devices receive their application after blessing (this is the default) and **not** after they are activated. This is a setting of the Production Device Group to which you have set your devices to be assigned upon blessing.

Please see [**How To Implement The Connected Factory Process**](https://developer.electricimp.com/manufacturing/factoryflow#application-firmware-loading) for more information.

**Note** Until a device has been activated, it has no agent. While the library can make use of a device’s agent in order that the agent URL can be sent to the BlinkUp mobile app, your code will need to gather this information after activation.

### Dependencies ###

This library requires the separate library [Bluetooth Firmware](https://github.com/electricimp/BluetoothFirmware):

```squirrel
#require "bt_firmware.lib.nut:1.0.0"
#require "btleblinkup.device.lib.nut:2.0.0"
```

One of the firmware versions included in [Bluetooth Firmware](https://github.com/electricimp/BluetoothFirmware) should then be included in your BTLEBlinkUp instantiation call:

```squirrel
// For imp004m
local bt = BTLEBlinkUp(serviceUUIDs, BT_FIRMWARE.CYW_43438);
```

```squirrel
// For imp006
local bt = BTLEBlinkUp(serviceUUIDs, BT_FIRMWARE.CYW_43455);
```

### Constructor: BTLEBlinkUp(*uuids, firmware[, lpoPin][, regonPin][, uart]*) ###

#### Parameters ####

| Parameter | Type | Required? | Description |
| --- | --- | --- | --- |
| *uuids* | Array of strings | Yes | Specify your application’s UUIDs for the BlinkUp service delivered by the library. See [**BLE Service UUIDs**, below](#ble-service-uuids) |
| *firmware* | String or blob | Yes | The required Bluetooth firmware for your device. See [**Dependencies**, above](#dependencies) |
| *lpoPin* | imp **pin** object | No | The imp GPIO pin connected to the Bluetooth LE radio’s LPO_IN pin. Default: **hardware.pinE** |
| *regonPin* | imp **pin** object | No | The imp GPIO pin connected to the Bluetooth LE radio’s BT_REG_ON pin. Default: **hardware.pinJ** |
| *uart* | imp **uart** object | No | The imp UART on which the imp’s Bluetooth radio is connected. Default: **hardware.uartFGJH** |

**Note** The last three parameters are required only on the imp004m. imp006-based applications should not supply arguments to these parameters; indeed, they will be ignored if you do.

#### BLE Service UUIDs ####

The constructor's *uuids* parameter is used to set UUIDs for the BlinkUp services delivered by the library. The UUIDs are supplied as a table which must contain the following keys:

| Key | UUID&nbsp;Role |
| --- | --- |
| *blinkup_service_uuid* | The BlinkUp service itself |
| *ssid_setter_uuid* | The BlinkUp service’s WiFi SSID setter characteristic |
| *password_setter_uuid* | The BlinkUp service’s WiFi password setter characteristic |
| *planid_setter_uuid* | The BlinkUp service’s Plan ID setter characteristic |
| *token_setter_uuid* | The BlinkUp service’s Enrolment Token setter characteristic |
| *blinkup_trigger_uuid* | The BlinkUp service’s BlinkUp initiation characteristic |
| *wifi_getter_uuid* | The BlinkUp service’s WiFi network list characteristic |
| *wifi_clear_trigger_uuid* | The BlinkUp service’s clear imp WiFi settings characteristic |

You must provide UUIDs for **all** of these keys or BlinkUp will fail and a runtime error with be thrown. Electric Imp reserves the right to add additional keys in future, so you should not extend this list yourself. If you wish your imp-enabled device to serve other GATT services alongside the BlinkUp service, this can be done using the [*serve()*](#serveotherservices) function detailed below.

The value of each key is a string.

**Note** The Bluetooth instance established by the constructor can be accessed by the host Squirrel using the instance’s *ble* property. You should always check that this instance is not `null` before performing any actions upon it.

## Class Methods ##

### listenForBlinkUp(*[advert][, callback]*) ###

This method provides the easiest way to provide BlinkUp via Bluetooth LE. It boots up the imp’s Bluetooth LE radio; sets up and serves two GATT services: BlinkUp and the standard Device Information service; prepares the imp to receive connections from the mobile app being used to provide the imp with activation data; and advertises the imp’s availability to other Bluetooth LE devices.

#### Parameters ####

| Parameter | Type | Required? | Description |
| --- | --- | --- | --- |
| *advert* | String or blob | No | The imp’s Bluetooth LE advertisement payload, up to 31 bytes in length. If you do not specify a payload, the library generates one for you using the BlinkUp service UUID (default or supplied) and the type of imp module as a device name string (eg. `imp004m`) |
| *callback* | Function | No | A function which will be triggered when a remote device connects to the imp, or actively disconnects from it |

The *callback* function has a single parameter of its own into which a table is passed containing any of the following keys:

| Key | Type | Notes |
| --- | --- | --- |
| *conn* | imp API BluetoothConnection instance | See [**hardware.bluetooth.onconnect()**](https://developer.electricimp.com/api/hardware/bluetooth/onconnect) |
| *address* | String | The hexadecimal address of the connecting device |
| *security* | Integer | The security mode of the connection (1, 3 or 4) |
| *state* | String | The state of the remote device: `"connected"` or `"disconnected"` |
| *error* | String | In the event of an error, a description of the error; otherwise `null` |

#### Return Value ####

Nothing.

#### Example ####

```squirrel
local bt = BTLEBlinkUp();
bt.listenForBlinkUp(null, function(data) {
    server.log("Device " + data.address + " has " + data.state);
});
```

### setAgentURL(*url*) ###

This method allows you to inform the instance of the URL of host device's agent. You can also set this directly via the object property *agentURL*. If set, the agent URL will be included in the device information service provided by the instance.

#### Parameters ####

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| *url* | Integer | No | The URL of the device's agent (Default: `""`) |

#### Return Value ####

String &mdash; the agent URL applied.

#### Example ####

```squirrel
local bt = BTLEBlinkUp();

// Register a handler to receive the agent URL from the agent
agent.on("set.agent.url", function(url) {
    // Set the agentURL property and configure BLE
    bt.setAgentURL(url);
    configureBLEBlinkUp();
}.bindenv(this));

// Ask the agent for its URL
// Agent simply returns its URL as a string
agent.send("get.agent.url", true);
```

### setSecurity(*mode, pin*) ###

This method applies the required security mode: 1, 3 or 4. For more information, please see [**hardware.bluetooth.setsecurity()**](https://developer.electricimp.com/api/hardware/bluetooth/setsecurity). If any errors are encountered, eg. an incorrect security mode has been requested, the method will report the fact and set the security level to the default: no security. For this mode, no value need be passed into *pin*, but modes 3 and 4 both require the provision of a six-digit decimal PIN code for pairing. This can be a string or an integer.

#### Parameters ####

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| *mode* | Integer | No | The required BLE security mode (Default: 1) |
| *pin* | Integer or string | No | The six-digit PIN code (Required for modes 3 and 4) |

#### Return Value ####

Integer &mdash; the security mode applied.

#### Example ####

```squirrel
local advert = "\x11\x07\x19\xD7\x68\xF3\x7C\xAF\xF2\xA5\xC9\x48\x55\xC4\xBE\x47\xDA\xFA\x0A\x09\x69\x6D\x70\x30\x30\x34\x2D\x42\x42";
local bt = BTLEBlinkUp();
bt.setSecurity(4, "163524");
bt.listenForBlinkUp(advert, function(data) {
    server.log("Device " + data.address + " has " + data.state);
});
```

### serve(*[otherServices]*) ###

This method sets up and begins serving BlinkUp and standard Device Information GATT services. The BlinkUp service uses the UUIDs supplied to the class constructor (or the defaults, if no UUIDs were specified). The latter uses the following UUIDs, as per the Bluetooth LE standard:

```squirrel
service = { "uuid": 0x180A,
            "chars": [
              { "uuid": 0x2A29, "value": "Electric Imp" },           // Manufacturer name
              { "uuid": 0x2A25, "value": hardware.getdeviceid() },   // Serial number
              { "uuid": 0x2A24, "value": imp.info().type },          // Model number
              { "uuid": 0x2A26, "value": imp.getsoftwareversion() }  // Firmware revision
            ] };
```

**Note** *serve()* is called implicitly by [*listenForBlinkUp()*](#listenforblinkupadvert-callback).

#### Parameters ####

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| *otherServices* | Array of tables | No | An array of service definition tables (or a single such service as a table) as detailed [here](https://developer.electricimp.com/api/hardware/bluetooth/servegatt). These are loaded and served alongside the two services mentioned above |

#### Return Value ####

Nothing.

#### Example ####

```squirrel
// Add Battery Level service
local bls = { "uuid": 0x180F,
              "chars": [
                { "uuid": 0x2A19,
                  "read": function(conn) {
                    local b = blob(1);
                    local battery_percent = hardware.vbat() / 4.2 * 100;
                    b.writen(battery_percent, 'b');
                    return b;
                  }.bindenv(this) }
              ] };

bt.serve(bls);
```

### advertise(*[advert][, min][, max]*) ###

This method advertises the imp to other Bluetooth LE-enabled devices using the argument passed into *advert*, or a self-generated advert based on the BlinkUp service UUID and the host imp’s type as its name, if *advert* is `null`. The parameters *min* and *max* are also optional, and default to 100 (milliseconds; the minimum and maximum advertising interval).

**Note** *advertise()* is called implicitly by [*listenForBlinkUp()*](#listenforblinkupadvert-callback).

#### Parameters ####

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| *advert* | String | No | The imp’s Bluetooth LE advertisement payload, up to 31 bytes in length. If you do not specify a payload, the library generates one for you using the BlinkUp service UUID (default or supplied) and the type of imp module as a device name string (eg. `imp004m`) |
| *min* | Integer | No | The minimum advertising interval in ms (Default: 100) |
| *max* | Integer | No | The maximum advertising interval in ms (Default: 100) |

#### Return Value ####

Nothing.

#### Example ####

```squirrel
local advert = "\x11\x07\x19\xD7\x68\xF3\x7C\xAF\xF2\xA5\xC9\x48\x55\xC4\xBE\x47\xDA\xFA\x08\x09\x69\x6D\x70\x30\x30\x34\x6D";
bt = BTLEBlinkUp();
bt.setSecurity(1);
bt.serve();
bt.advertise(advert, 90, 110);
server.log("Bluetooth running...");
```

### onConnect(*[callback]*) ###

This method registers a callback function that will be triggered when a remote device connects to the imp, or manually disconnects from it.

**Note** *onConnect()* is called implicitly by [*listenForBlinkUp()*](#listenforblinkupadvert-callback).

#### Parameters ####

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| *callback* | Function | No | A function which will be triggered when a remote device connects to the imp, or actively disconnects from it |

The *callback* function has a single parameter of its own into which a table is passed containing any of the following keys:

| Key | Type | Notes |
| --- | --- | --- |
| *conn* | imp API BluetoothConnection instance | See [**hardware.bluetooth.onconnect()**](https://developer.electricimp.com/api/hardware/bluetooth/onconnect) |
| *address* | String | The hexadecimal address of the connecting device |
| *security* | Integer | The security mode of the connection (1, 3 or 4) |
| *state* | String | The state of the remote device: `"connected"` or `"disconnected"` |
| *error* | String | In the event of an error, a description of the error; otherwise `null` |

#### Return Value ####

Nothing.

## Release Notes ##

- 2.0.0
    - Support imp006.
- 1.0.0
    - Initial public release.

## License ##

This library is licensed under the terms and conditions of the [MIT License](LICENSE).
