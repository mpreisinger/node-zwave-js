# Device configuration files

Since older versions of the Z-Wave standard don't allow us to request all the information we need from the devices themselves, there is a need for configuration files. These are located under `config/devices/<manufacturerID-as-hex>/<device-name>[_<firmware-range>].json`.

## Properties

The following properties are defined and should always be present in the same order for consistency among the config files:

| Property            | Description                                                                                                      |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `manufacturer`      | The name of the manufacturer                                                                                     |
| `manufacturerId`    | The ID of the manufacturer (as defined in the Z-Wave specs) as a 4-digit hexadecimal string.                     |
| `label`             | A short label for the device                                                                                     |
| `description`       | A longer description of the device, usually the full name                                                        |
| `devices`           | An array of product type and product ID combinations, [see below](#devices) for details.                         |
| `firmwareVersion`   | The firmware version range this config file is valid for, [see below](#firmwareVersion) for details.             |
| `associations`      | The association groups the device supports, [see below](#associations) for details.                              |
| `supportsZWavePlus` | If set to `true`, the device complies with the Z-Wave+ standard. In this case, omit the `associations` property. |
| `proprietary`       | A dictionary of settings for the proprietary CC. The settings depend on each proprietary CC implementation.      |
| `paramInformation`  | A dictionary of the configuration parameters the device supports. [See below](#paramInformation) for details.    |

### `devices`

Each device in the Z-Wave standard is identified by its product type and product ID. A config file that is valid for both `0x0123 / 0x1000` and `0x2345 / 0x0001` would have the following `devices` entry:

```json
"devices": [
	{
		"productType": "0x0123",
		"productId": "0x1000"
	},
	{
		"productType": "0x2345",
		"productId": "0x0001"
	}
]
```

### `firmwareVersion`

Since different firmware versions of a device may have different config params, you must specify the firmware range for each config file. A config file that is valid from version `2.0` to `4.75` would have the following `firmwareVersion` entry:

```json
"firmwareVersion": {
	"min": "2.0",
	"max": "4.75"
}
```

The default `min` version is `0.0` and the default `max` version is `255.255`.
All other firmware ranges should be reflected in the filename.

### `associations`

For devices that don't support the Z-Wave+ standard, the associations must be defined. The property looks as follows:

```json
"associations": {
	// One entry for each association group
	"1": {
		"label": "Label for group #1", // required
		"maxNodes": 5 // How many nodes may be in that group, required
	},
	"2": {
		"label": "Label for group #2",
		"description": "A description what group #2 does", // optional
		"maxNodes": 1, // SHOULD be 1 for the lifeline, some devices support more nodes
		"isLifeline": true, // Whether this is the Lifeline group. SHOULD exist exactly once, some nodes require more groups to report everything
		"noEndpoint": true, // Whether node id associations must be used for this group, even if the device supports endpoint associations, (optional)
	},
	// ... more groups ...
}
```

The `isLifeline` key is used to determine which group sends the controller device status updates. It may only be defined for one group, which also must have a `maxNodes` of 1.

### `paramInformation`

This property defines all the existing configuration parameters. It looks like this

```json
"paramInformation": {
	"1": { /* parameter #1 definition */},
	"2": { /* parameter #2 definition */},
	// ... more parameters ...
}
```

where each parameter definition has the following properties:

| Parameter property | Type    | Required? | Description                                                                                                                                                              |
| ------------------ | ------- | :-------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `label`            | string  |    yes    | A short name for the parameter                                                                                                                                           |
| `description`      | string  |    no     | A longer description what the parameter does                                                                                                                             |
| `valueSize`        | number  |    yes    | How many bytes the device uses for this value                                                                                                                            |
| `minValue`         | number  |    yes    | The minimum allowed value for this parameter                                                                                                                             |
| `maxValue`         | number  |    yes    | The maximum allowed value for this parameter                                                                                                                             |
| `unsigned`         | boolean |    no     | Whether this parameter is interpreted as an unsigned value by the device (default: `false`). This simplifies usage for the end user.                                     |
| `defaultValue`     | number  |    yes    | The factory default value of this parameter.                                                                                                                             |
| `readOnly`         | boolean |    no     | Whether this parameter can only be read                                                                                                                                  |
| `writeOnly`        | boolean |    no     | Whether this parameter can only be written                                                                                                                               |
| `allowManualEntry` | boolean |    yes    | Whether this parameter accepts any value between `minValue` and `maxValue`. If `false`, `options` can must be used to specify the allowed values.                        |
| `options`          | array   |    no     | If `allowManualEntry` is `false`, this property must contain an array of objects of the form `{"label": string, "value": number}`. Each entry defines one allowed value. |

### Partial parameters

Some devices use a single parameter number to configure several, sometimes unrelated, options. For convenience, `node-zwave-js` provides a simple way to define these values as multiple (partial) configuration parameters.

For example,

```json
"40[0x01]": {
	"label": "Button 1: behavior",
	/* parameter definition */
},
"40[0x02]": {
	"label": "Button 1: notifications",
	/* parameter definition */
},
"40[0x04]": {
	"label": "Button 2: behavior",
	/* parameter definition */
},
"40[0x08]": {
	"label": "Button 2: notifications",
	/* parameter definition */
},
```

defines 4 partial parameters that each switch a single bit of parameter #40. Using the appended bit mask (e.g. `[0x01]`), you can configure which bits each partial parameter affects.

Partial parameters must follow these rules:

1. Each partial parameter must have the same `valueSize`
1. Each bit mask must fit into the configured `valueSize` of the parameter.
1. The `minValue`, `maxValue` and `defaultValue` as well as options values are relative to the lowest bit the bit mask. If the bit mask is `0xC` (binary `1100`), these properties must be in the range 0...3 (2 bits). Any required bit shifts are automatically done.

## Contributing configuration files

In order to get your configuration file included in this library, two things must be done:

1. Check your file for potential problems using `npm run lint:config`. Warnings in your file may be tolerated if there is a good reason for them. Errors must be fixed.
2. Add your file to the index using `npm run config index`.
3. Create a PR.
