# Musical Event Notation Specification

This is a proposal of Musical Event Notation Specification for offline edits.<br>

Musical Event Notation is designed for belows:

1. Storing list of events within user interfaces such as piano roll and tracker
2. Exchanging notations between user interfaces and custom note/expression transformer such as arpeggiator and quantizer.
3. The common notation to be played back by Playback Node in the Event Control Layer

Musical Event Notation is NOT designed for belows:

:x: realtime event messages (but it CAN be used for message body of API)<br>
:x: file format like Standard MIDI File (SMF)

## Data Types

- integer: integer value and CAN be minus value
- float: number including decimal value
- strings: represented by `" "`, the text
- List: represented by `[ ]` in JSON which has values with order.
- Map: represented by `{ }` in JSON which has key-value pairs. Each key MUST be string.

## General Structure

The top level of Musical Event Notation is a Map which has elements below.

|Item|Data Type|Description|
|---|---|---|
|ppq|integer|Pulses Per Quarter, the number of ticks per quarter beat|
|notes|List|see [Note](#note) section|
|params|List|see [Param](#param) section|

Every Musical Event Notation MUST have ppq and whether notes or params.

Simple example:
``` json
{
    "ppq": 960,
    "notes": [ { "tick": 240, "pitch": 64, "duration": 480 } ],
}
```

Musical Musical Event Notation CAN have meta information such as track name and channel number like below.
``` json
{
    "ppq": 960,
    "notes": [ /* ... */ ],
    "params": [ /* ... */ ],
    "trackName": "Lead Guitar"
}
```

## Note
Note represents one note event.<br>
Notes are special events to represent

|Item|Data Type|Description|
|---|---|---|
|tick|integer|the note on in ticks|
|pitch|integer|the note number (0-127)|
|duration|integer|the length of a note in tick|
|velocity|integer|0-127|
|off_velocity|integer|(optional) note off velocity (0-127)|

Each element of `notes` list MUST be ordered by `tick`.<br>
A Note event MUST have `tick` and `number`.<br>
If duration or velocity is not provided, nodes CAN set default values.

Note event CAN have additional information such as `channel` for MPE or per-note parameter.

## Param
Param represents one parameter change event.<br>
|Item|Data Type|Description|
|---|---|---|
|path|string|see [Path](#path-of-parameter) section|
|points|List|see [Point](#Point) section|

Simple example: 
``` json
"params": [
    {
        "path": "midi.cc.1",
        "points": [ { "tick": 240, "value": 127 } ],
    }
]
```

Param CAN have supplemental attributes such as the definition of enum values like below.

``` json
"params": [
    {
        "path": "synth1.lfo",
        "points": [ "..." ],
        "type": "enum",
        "enum": [
            { "value": 0, "name": "Off" },
            { "value": 1, "name": "On" }
        ]
    }
]
```

## Path of Parameter
Path represents the semantic of a parameter and it MUST be like `domain.parameter`.
The path SHOULD be connected by `.`.

The domain is generally from an element which the node has.
i.e. `midi`, `gui`, `transport`, `sequencer`

If the node handles specific DSP parameters, path would be like `synth1.lfo.speed`.

## Point
A Point represents a single value point and the interpolation between points.

|Item|Data Type|Description|
|---|---|---|
|tick|integer|the change or beginning of the transition in ticks|
|value|integer or float||
|curve|string|(optional) interpolation curve between the point and the next one|

Values MUST have `tick` and `value`.
`tick` of each point MUST be identical and ordered within the list (ascending).<br>

`value` CAN represent enum, but it MUST be integer value as the index and it SHOULD start with 0.

Each node SHOULD implement `step` (jumping to the value) and `linear` of the curve attribute.
Value CAN have additional `curve` types, but it SHOULD be continuous.<br>
If `curve` is not specified, it MUST be treated as `step`.<br>
If `curve` is unknown to the node, it SHOULD be treated as `linear`.

Value CAN have supplemental attributes for `curve` like below.
``` json
"params": [
    {
        "path": "midi.pitchbend",
        "points": [
            { "tick": 0, "value": 0 },
            { "tick": 240, "value": 8191, "curve": "exp-curve", "exp": 0.5 },
            { "tick": 480, "value": 0 },
        ]
    }
]
```

## References
[RFC 8259 - The JavaScript Object Notation (JSON) Data Interchange Format](https://datatracker.ietf.org/doc/html/rfc8259)
