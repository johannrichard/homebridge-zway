# homebridge-zway

...is a Homebridge module for [Z-Way Server](http://razberry.z-wave.me/index.php?id=24).

This platform lets you bridge a Z-Way Server instance (for example, running on [RaZBerry](http://razberry.z-wave.me) hardware or with a [UZB1](http://www.z-wave.me/index.php?id=28)) to HomeKit using Homebridge.

Homebridge requires Z-Way Server version 2.0.1 or greater, and has so far only been tested against 2.0.1 (though it is expected to work with 2.1.1).

## Quick Start

1. `sudo npm install -g homebridge` (See the [Homebridge](https://github.com/nfarina/homebridge) project site for more information)
2. `sudo npm install -g homebridge-zway`
3. Edit `~/.homebridge/json.config` and add the following:

```
"platforms": [
    {
        "platform": "ZWayServer",
        "url": "http://your.ip.goes.here:8083/",
        "login": "[admin]",
        "password": "[password]"
    }
]
```

Then see the [Configuration](#configuration) and [Tags](#tags) sections below to to customize the bridge for your environment or devices.

## Supported Devices

Support is currently designed around Z-Wave devices. The bridge uses the `VDev` interface, so other device types (such as EnOcean) should also work, but this has not been tested.

Generally speaking, the following types of devices will work:

* On/off switches
* Dimmers
* RGB bulbs (e.g. Aeon Labs')
* Thermostats (heating only for now!)
* Temperature sensors
* Door/window sensors
* Light sensors (needs work)
* Motion sensors
* :tada: :new: Door Locks (e.g. Danalock)
* :tada: :new: Relative Humidity sensors (e.g. Aeon Labs Multisensor 6...needs testing!)

Additional devices in progress:

* Water/contact sensors
* Remotes/buttons (maybe...looks tricky)

## Problems/Troubleshooting

If you have a problem with the Z-Way Server bridge or with a particular device, please create an Issue and attach the contents of `/accessories` from your Homebridge instance, and of `/ZAutomation/api/v1/devices` from your Z-Way Server instance (or at least those parts that pertain to your problem).

# Configuration

## Required

The minimum configuration includes the following 3 parameters

    {
        "platform": "ZWayServer",
        "url": "http://192.168.1.100:8083/",
        "login": "[admin]",
        "password": "[password]"
    }

| Key | Description |
| --- | --- |
| `url` | The base URL of your Z-Way Server installation. For instance, the URL above would be the right pattern if your Z-Way web UI is accessed at `http://192.168.1.100:8083/smarthome/#/elements`. Since the protocol, address, port, and path are all used, you're covered if you change any of them (i.e. if you're running Z-Way Server behind a reverse-proxy). |
| `login` | A username with permissions in Z-Way. Using `admin` is actually not recommended, but you can to start with it if you want to make sure everything's working. It's best to create another user so you don't have your admin password sitting in the configuration file. |
| `password` | The password for the user specified in `login`. |

## Optional

The following additional configuration options are supported

| Key | Default | Description |
| --- | :---: | --- |
| `poll_interval` | `2` | The time in seconds between polls to Z-Way Server for updates. 2 seconds is what the Z-Way web UI uses, so this should probably be sufficient for most cases. |
| `battery_low_level` | `15` | For devices that report a battery percentage, this will be used to set the `BatteryLow` Characteristic to `true`. |
| `split_services` | `false` | **DEPRECATED** This setting affects how Characteristics are organized within an accessory. For instance, if set to "true", the `BatteryLevel` and `StatusLowBattery` Characteristics were put into a `BatteryService`, where the default of `false` caused them to be simply added as additional Characteristics on the main Service. This was done mainly to support the Eve app better, which made separate Services appear the same as whole different Accessories. The Eve app now groups services in the same accessory, so this will probably be changed to default to `true` and later be removed entirely. |
| `opt_in` | `false` | If this is set to `true`, only devices tagged with `Homebridge.Include` will be bridged. This is mainly useful for development or troubleshooting purposes, or if you really only want to include a few accessories from your Z-Way server. |

# Tags

You can change the default behavior of Homebridge by adding certain tags to your devices in the Z-Way web UI. Sometimes this may be necessary to get certain devices to be bridged properly, as there are a large number of Z-Way devices and sometimes the "guessing" that Homebridge does to get one device right may be the wrong answer for a different device.

Tags are case sensitive. Some tags allow you to specify a value, and these have the value after a `:` character (everything after the `:` is the value). Tags without a value are boolean, and are tested for presence or absence.

#### Homebridge.Skip

Any devices with this tag will not be bridged, and will be excluded from the logic used by Homebridge to try and translate Z-Way devices to HomeKit devices.

#### Homebridge.Include

Only used in conjunction with the `opt_in` configuration option above, this marks a device to be included in opt-in mode, while all other devices are skipped.

#### Homebridge.IsPrimary

This overrides or supplements Homebridge's logic for figuring out what kind of Accessory or Services to build from your Z-Way devices, and sometimes how to use the multiple devices within an Accessory. This is particularly useful if Homebridge gets it wrong by default, or the components of your device are unusual or ambiguous.

For instance, if you have a Devolo Door/Window sensor but are primarily using it for a temperature sensor, you could tag the temperature sensor device with `Homebridge.IsPrimary`, which would change the way the device is reported to HomeKit.

For another example, the Aeon Labs RGB Bulb has three dimmers: one for the color LEDs, one for "Cold" white and one for "Soft" white. It's not obvious to Homebridge which of the dimmers controls the color LEDs, but it can figure out that it's an RGB bulb. So, you should put `Homebridge.IsPrimary` on the dimmer for the color LEDs, and the other dimmer devices will be treated as extras.

#### Homebridge.Accessory.Id:*value*

Manually specifies the Accessory identifier to use for this device. This has the effect of allowing you to split or merge devices that would be grouped differently by default by Homebridge's translation logic.

For instance, many Z-Wave devices include a temperature sensor that has nothing to do with their primary function (such as an outlet switch), so you could give that temperature sensor a different Accessory Id, and it would appear to HomeKit as a separate Accessory. Or, if you have a Danfoss Living Connect thermostat (which does not report the room temperature via Z-Wave) and a temperature monitoring device in the same room, you could give them the same ID and Homebridge would bridge them as a single device on the HomeKit side.

#### Homebridge.Characteristic.Description:*value*

This tag lets you override the description for the Characteristic(s) created from this device. This may affect the way the Characteristic is displayed in your HomeKit app.

#### Homebridge.Service.Type:*value*

This tag allows you to explicitly specify what kind of HomeKit Service to create for an accessory. This is only supported for a specific set of cases, and even then may break your bridge! It will only be effective on the primary device of a Service, so either on the primary device of an Accessory or on a device like the Aeon Labs RGB bulb which has multiple dimmers which have to be split off into their own Services.

##### Make a Switch show as a `Lightbulb`

Tagging a device with `Homebridge.Service.Type:Lightbulb` allows you to explicitly report a `switchBinary` as a HomeKit `Lightbulb` (normally only `switchMultilevel`s will be automatically bridged as lights). This means that if you ask Siri to "turn off the lights" in a room, the marked device should be included.

##### Change a Dimmer to a Switch

Somewhat the opposite of above, you can specify `Homebridge.Service.Type:Switch` on a dimmer to treat that device as a standard switch instead of a dimmer. Besides doing this just out of preference, this is handy on the aforementioned Aeon Labs RGB bulb's extra "white" dimmers, because the primary (color) dimmer actually controls the dimming of the two whites.

##### `sensorBinary` as Contact or Motion sensor

A contact sensor or motion sensor may only be reported by Z-Way as a `sensorBinary`, which is too vague to determine its purpose. In this case you must specify `Homebridge.Service.Type:MotionSensor` or `Homebridge.Service.Type:ContactSensor` so that the bridge will know how to report the device. See also `Homebridge.Characteristic.Type` below.

##### Door/Window Sensor Service

There is not really a direct analogue to a `Door/Window` sensor in HomeKit--the possibilities are `GarageDoor`, `Door`, `Window`, and `ContactSensor`. The first three are really designed for door or window *controls* instead of sensors, and then the last one is a very generic type of sensor. Homebridge now lets you choose which of these four to use for your sensor by specifying one of the following values in this tag:

* `GarageDoor`: This is the old default, and may make the most sense if you have an apartment with a single door. It reports the door as "Open" or "Closed" in most apps, and you get iOS notifications when the door opens or closes. However, if you have an actual garage door opener that is controllable by Siri, when you say "Open the Garage Door," Siri will try to also open your door sensor...which will fail, and she will complain, which can be annoying. It also adds a required "ObstructionDetected" sensor, which does nothing.
* `Door`: This is the new default and avoids the "multiple" Garage Door problem above. However it reports position in percent (always 0% for closed or 100% for open), and still generates iOS notifications when the door opens and closes. It will also have a "PositionState" characteristic, which is always "Stopped".
* `Window`: This is identical to `Door`, but may be categorized differently by Siri or in apps.
* `ContactSensor`: This treats the `Door/Window` sensor as a simple contact sensor (on or off). This has two main advantages:
  a. It's the simplest option, and doesn't have any superfluous, non-working characteristics.
  b. You don't get any iOS notifications for state changes, so pick this if you find those annoying.
  But, you won't be able to ask Siri about the state of the "Door", and app support for this characteristic has been historically lacking (Eve works great now). Also note that you can invert the value with `Homebridge.ContactSensorState.Invert`, which may result in a more intuitive value being shown.

##### More

There will be additional devices and use cases where this will be used. If you think you have a good use case for this that is not supported by the current code, please submit an issue with the guidelines above.

#### Homebridge.Characteristic.Type:*value*

Like [`Homebridge.Service.Type`](#homebridgeservicetypevalue), this allows you to explicitly define the type of Characteristic that will be created for a given device. This will override the bridge's own logic for selecting which Characteristic(s) to build from a device, so use it with caution!

This tag is particularly useful for scenarios where the physical device is reported ambiguously by Z-Way. For instance, the Vision ZP 3012 motion sensor is presented by Z-Way merely as two `sensorBinary` devices (plus a temperature sensor), one of which is the actual motion sensor and the other is a tampering detector. The `sensorBinary` designation (with no accompanying `probeTitle`) is too ambiguous for the bridge to work with, so it will be ignored. To make this device work, you can tag the motion sensor device in Z-Way with `Homebridge.Characteristic.Type:MotionDetected` and (optionally) the tamper detector with `Homebridge.Characteristic.Type:StatusTampered`. (Note that for this device you will also need to tag the motion sensor with `Homebridge.Service.Type:MotionSensor` and `Homebridge.IsPrimary`, otherwise the more recognizable temperature sensor will take precedence.)

#### Homebridge.ContactSensorState.Invert

If you have a `ContactSensor`, this will invert the state reported to HomeKit. This is useful if you are using the `ContactSensor` Service type for a `Door/Window` sensor, and you want it to show "Yes" when open and "No" when closed, which may be more intuitive. The default for a `ContactSensor` is to show "Yes" when there is contact (in the case of a door, when it's closed) and "No" when there is no contact (which for a door is when it's open).

# Technical Detail

## Bridging and Mapping Logic

As with any bridge, the devices on the The Z-Way Server side do not necessarily map perfectly to the HomeKit device model. So, the Z-Way bridge does a lot of guesswork and generalization to make the devices make sense on the HomeKit side.

### Recombination

A Z-Wave device usually supports multiple Command Classes for different kinds of controls and sensors, and Z-Way splits all of those devices up into separate "Virtual Devices". This is the most flexible way to deal with composite devices, but having all those individual components in a HomeKit app as separate Accessories would be very clumsy, and sometimes impossible (for instance, a HomeKit Thermostat requires a temperature sensor device in the same Accessory and Service, so it can't be bridged by itself). So, Homebridge attempts to recombine those Virtual Devices back into a single composite device.

### Primary Devices

After combining the sub-devices back into a single composite device, it then tries to make sense of what kind of Accessory that composite device should be in HomeKit. It does this by trying to determine which sub-device should be treated as the "primary" device.

This can be tricky, since devices may contain any combination of sensors and controls. The bridge tries to pick more unusual or specific sub-devices out first. For example, many many devices include a temperature sensor, but a Door/Window sensor is far less common, and if present probably indicates the device's intended main purpose.

### Service Type

Once Homebridge decides on the primary device, it chooses a Service type that corresponds to that device and attempts to configure that Service. For example, if it finds a thermostat device in Z-Way, it will create a Thermostat Service in HomeKit and look for a temperature sensor device in the same composite device.

### Required and Optional Characteristics

Many Services in HomeKit have a set of Characteristics that must appear in the Service to be in compliance--for example the aforementioned Thermostat, which requires a thermostat device and a temperature sensor. Homebridge will attempt to fill all the requirements from the composite device. If it fails, it will not bridge that Service (and likely nothing else in the device)! You may see messages to this effect at the console on startup.

After fulfilling the required Characteristics, the bridge will look at the remaining sub-devices and--if it understands them and knows what Characteristic(s) it can build from them, it will tack those Characteristics onto the primary service. HomeKit seems fairly flexible about this sort of tack-on extras ability, so if you have a device that is an outlet switch with a temperature sensor in it, you'll get a Switch with a TemperatureSensor in HomeKit as well.

### Additional Services

Because HomeKit does not support more than one of the same Characteristic in the same Service, if the bridge encounters a composite device with this composition (say, multiple dimmers) it will in some cases build additional services to contain the additional Characteristics.

*NOTE: At the moment, the only scenario where this happens is RGBW bulbs.*

If this happens, you may see ambiguously titled controls in your HomeKit app (multiple Brightness controls, for instance).
