# Musical Node Model Proposal for Distributed Audio Systems (draft)

This is a proposal for Musical Node Model.

The content of this repository MAY change in the future.<br>
If you have any suggestions or similar projects, please feel free to submit them to the [Issues](https://github.com/h1data/musical-node-model/issues) or [contact the author](https://h1data.github.io/contact/).

This proposal has three main goals;
1. To separate digital sound systems into layers; UI, event handling, and DSP

2. To define musical notation and realtime event message semantics for communications between musical nodes

3. To establish node graph architecture that allows independent programs to be connected as musical nodes

## Motivation

Unlike hardware modular synthesizers, software sound systems generally have a single I/O defined by the operating system, and a single program handles everything.

``` mermaid
flowchart TD
  subgraph OS ["Operating System"]
    direction TB
    subgraph Program ["*Single* Sound Program"]
      Daw["User Interfaces<br>i.e. Sequencer, Event List, Piano Roll, Tracker, MML, or Codes<br>+<br>Synthesizers, Effecters"]
    end
    audio-in{{Audio In}}
    audio-out{{Audio Out}}
    midi-in{{MIDI In}}
    midi-out{{MIDI Out}}
    midi-in --> Program --> midi-out
    audio-in --> Program  --> audio-out
  end
  ex-in{{Hardware Input}}
  ex-out{{Hardware Output}}
  ex-in --> OS --> ex-out
```
This ecosystem has several problems;

1. Monolithic<br>
Various applications represent kinds of user interfaces such as piano roll, tracker, and multi-channels for audio and MIDI I/O, however, they are all packaged into one application.
Users cannot flexibly combine A's interface, B's synth engine, and C's effecter.

2. Ambiguous Boundaries of Host and Plugins<br>
Plugin ecosystems such as VST represents to use synthesizers and effecters in different applications.<br>
However, we have been experienced compatibility issues between plugins and host applications, CPU architectures (x86/arm and 32/64 bit), and operating systems. Who took on those responsibility?

3. Weak Integration between Event Messages and Audio Streams<br>
Existing protocols such as [MIDI](https://midi.org/midi-1-0-detailed-specification) and [OSC (Open Sound Control)](https://opensoundcontrol.stanford.edu/spec-1_0.html)
are designed for communication between external instruments or devices.
However, it lacks tight interaction between event streams and audio processing, such as parameter modulation, transport synchronization, and scheduled events.

## Approach

The Musical Node Model's approach is to separate a digital sound system into the three layers.<br>

1. Notation Layer<br>
User interface layer that handles inputs from the GUI and stores/transforms notes and expression events as the [Musical Event Notation](NOTATION.md).<br>
There have been de facto standards of user interfaces such as an event list of MIDI, piano roll, tracker, or MML texts.<br>
Event Transformer Node modifies Musical Event Notation offline,
while Event Effecter Node in Event Control Layer processes realtime event streams.

2. Event Control Layer<br>
This layer that transports musical notes and expressions, and communicates with the DSP Layer and external devices.
Realtime event effecters such as arpeggiator and note echo also belong to this layer.<br>
This layer MAY use MIDI 2.0 concepts such as MIDI-CI and UMP, however, aimed to adopt with MIDI 1.0 in draft.

3. DSP Layer<br>
Synthesis or effects for audio signals. i.e. synthesizers, effecters, or mixers.<br>

A "Node" is an independent processing unit that exchanges musical events or audio streams with other nodes in the system.
A node MAY run in a separate program or inside the same process (as a library).

An entire digital sound system by Musical Node Model is shown below.
``` mermaid
flowchart TB
  subgraph OS ["Operating System"]
    direction TB
    subgraph System ["Digital Sound System"]
      direction TB
      subgraph notation-layer ["Notation Layer"]
        direction LR
        Interface["User Interface Node<br>i.e. Sequencer or Tracker or Event List or Piano Roll"]
        <-- Musical Event Notation -->
        eventTransformer["Event Transform Node<br>i.e. Arpeggiator or Quantizer"]
      end
      subgraph event-layer ["Event Control Layer"]
        direction LR
        event-in{{"External Event In"}} -- "MIDI, OSC" --> recorder["Recorder Node"]
        -- "Musical Event Notation" --> toNotation{{"to Notation Layer"}}
        recorder 
        transporter["Transport Control Node"] -- "Transport Signal" -->
        playback["Playback Node"]  -- "Realtime Event Messages" -->
        event-effecter["Event Effecter Node<br>i.e. Arpeggiator, note echo (in realtime)"] -- "Realtime Event Messages"
        --> to-dsp{{"to DSP Layer"}}
        event-effecter -- "Realtime Event Messages" --> event-out{{to External Event Out}}
        transporter -- "Transport Signal" --> recorder
        from-notation{{"from Notation Layer"}} -- "Musical Event Notation" --> playback
      end
      subgraph dsp-layer ["DSP Layer"]
        direction LR
        from-event{{"from Event Layer"}} -- "Realtime Event Messages" -->
        generator["Signal Generator<br> i.e. gate, trigger signal from parameters"] -- "Audio Signal" -->
        synth["Synthesizer Node"] -- "Audio Signal" --> Effecter["Effecter Node"]
        audio-in{{Audio In}} -- "Audio Signal" --> Effecter -- "Audio Signal" --> audio-out{{Audio Out}} 
        generator -- "Audio Signal" --> Effecter
      end
      notation-layer <-- "Musical Event Notation" -->
      event-layer -- "Realtime Event Messages" -->
      dsp-layer
    end
    pc-audio-in{{Audio In}}
    pc-audio-out{{Audio Out}}
    pc-midi-in{{MIDI In}}
    pc-midi-out{{MIDI Out}}
    pc-midi-in --> System --> pc-midi-out
    pc-audio-in --> System --> pc-audio-out
  end
  ext-in{{Hardware Input}}
  ext-out{{Hardware Output}}
  ext-in --> OS --> ext-out
```

The Nodes in Event Control Layer and DSP Layer SHOULD be managed with directed acyclic graph (DAG) model.
However, the internal structure of Notation Layer is NOT restricted to a specific graph model because it is not realtime.

## Roadmaps

The following are what to define in the project:

- [ ] Musical Event Notation Structure<br>
Designed to represent communications between nodes in Notation Layer and Event Control Layer for event in .<br>See [NOTATION.md](NOTATION.md) for the details.<br>

- [ ] Playback Node<br>
The Playback Node WILL parse Musical Event Notation and send realtime event messages according to the transport signal from Transport Control Node.

- [ ] Transport Control Node<br>
The Transport Control Node WILL send transport signal for the start, stop, and sync of event playback between nodes in Event Control Layer.<br>
It MAY use the MCU protocol or DAW Control Profile of MIDI 2.0.

- [ ] Real Time Event Message Structure<br>
The real event messages WILL be used to communicate between nodes in Event Control Layer and based on Musical Event Notation, that includes timestamp and transition curve of parameters.<br>
It MAY use MIDI 2.0 UMP.

- [ ] Signal Generator Node<br>
The Signal Generator Node WILL generate audio signals from realtime event messages like gate, trigger, and sample and hold in hardware modular synthesizers.
This node bridges the Event Control Layer and the DSP Layer by converting musical events into modulation signals.

- [ ] (optional) Audio Buffer Broker<br>
The Audio Buffer Broker WILL manage the connection between nodes in DSP Layer by audio bus.<br>
There are already Core Audio, ASIO, and JACK, which manage audio signals between different threads or programs. Also, WebAudio API and AudioWorklet represent audio graph node systems.