# klang | C++ for audio

## Getting Started

To use klang, simply include the **klang.h** header file in your C++ project.

```cpp
#include <klang.h>
using namespace klang::optimised; // optional
```

Audio objects are then accessible using the `klang` namespace (e.g. Effect, Sine). klang tries to use plain language to describe objects, so the namespace is used to avoid conflict with similar terms in other APIs or code. If this is not needed, add the `using namespace` directive as shown above.

The core data type in klang is `signal`, an extension of the basic C type `float` with additional audio and streaming functionality. In many cases, signals can be used interoperably with floats, facilitating integration of klang with other C++ code, while enabling signal-flow expressions, such as in the following wah-wah effect ...

```cpp
// signal in, out;
// Sine lfo;
// LPF lpf;

signal mod = lfo(3) * 0.5 + 0.5;
in >> lpf(mod) >> out;
```

... where `lpf` is a low-pass filter, `in` is the input signal, `out` the output signal, and `mod` is a signal used to modulate the filter's parameter (~cutoff frequency), based on a low-frequency (3Hz) sine oscillator (`lfo`).

DSP components, like oscillators or filters, are declared as objects (using struct or class) as either a `Generator` (output only) or `Modifier` (input / output), supplying functions to handle parameter setting (`set()`) and signal processing (`process()`) ...

```cpp
struct LPF : Modifier {
   param a, b;

   void set(param coeff) {
      a = coeff;
      b = 1 - a;
   }

   void process() {
      (a * in) + (b * out) >> out;
   }
};
```

Here, the `Modifier` parent class adapts the LPF code so it can be used as before. Parameters have type `param`, which is derived from `signal`, allowing constants, floats, or signals to be used interchangeably with or as parameters. Code may use either signal (<<, >>) or mathematical (=) operators interchangeable, to allow you to express the audio process in a manner best suited to the context or other reference material. Filters are often described in mathematical terms, so you could also write: `out = (a * in) + (b * out);`.

More complex audio processes are created by combining klang objects, as in this simple subtractive synthesiser:

```cpp
struct Subtractive : Note {
   Saw osc;
   LPF lpf;
   Sine lfo;

   event on(Pitch pitch, Amplitude velocity) {
      const Frequency frequency(pitch -> Frequency);
      osc.set(frequency);
   }

   void process() {
      signal mod = lfo(3) * 0.5 + 0.5;
      osc >> lpf(mod) >> out;
   }
};
```

This class supplies the `Note` definition to be used as part of a synthesiser (`Synth` - see the Examples linked below). It processes audio, but also handles events such as a **note on**, where the supplied pitch is converted to a frequency (in Hz) for use in the oscillator. By default, without code to handle a **note off**, the note and audio processing will be automatically terminated when one is received.

But most instruments continue making sound after a note is 'released'...

```cpp
struct Subtractive : Note {
   Saw osc;
   LPF lpf;
   Sine lfo;
   ADSR adsr;

   event on(Pitch pitch, Amplitude velocity) {
      const Frequency frequency(pitch -> Frequency);
      osc.set(frequency);
      adsr.set(0.25, 0.25, 0.5, 5.0);
   }

   event off() {
      adsr.release();
   }

   void process() {
      signal mod = lfo(3) * 0.5 + 0.5;
      osc * adsr >> lpf(mod) >> out;
      if (adsr.finished())
         stop();
   }
};
```

This example shapes the note and adds a release stage using an ADSR amplitude envelope. The `ADSR` is a type of `Envelope` that takes four parameters (attack, decay, sustain, release - set in `on()`) and uses its output to scale the (\*) amplitude of the signal. To add the 'release' stage and continue processing audio after a **note off**, we add the `off()` event and trigger the release of an ADSR envelope. Now, `process()` will continue to be called until you tell it to `stop()`, which we call when the ADSR is finished (that is, `adsr.finished()` is true).

## Usage in a C++ project

This object defines the processing for a single synth note that can then be used in any audio C++ project, placing the following code fragments at appropriate points in your code (e.g. myEffect/mySynth mini-plugin or any AU/VST plugin, JUCE app, etc.):

```cpp
  klang::Subtractive note;

  note.start(pitch, velocity);   // Note On
  note.release(pitch, velocity); // Note Off

  klang::Buffer buffer = { pfBuffer, numSamples };
  if(!note.process(buffer))
     note.stop();
```
