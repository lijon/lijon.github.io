---
title: AUv3 Parameters
layout: default
---

This article will give you some useful hints regarding AUParameter implementation. It's mainly written for plugin developers.

# Parameter flags

## Writable

My host app AUM only lists the parameters that are marked as writable.
So make sure to add `kAudioUnitParameterFlag_IsWritable` to your `param.flags` for all parameters that you want to expose for external control.

## Linear vs Logarithmic

Another flag worth mentioning is `kAudioUnitParameterFlag_DisplayLogarithmic`, which is suitable for frequency, ADSR time values, etc.
AUM checks this flag and if set it maps the linear input value to the param like this:

    paramValue = pow(maxValue/minValue, inValue) * minValue
    
_Note that a logarithmic parameter can never have values <= 0!_

If the flag is not set, it treats the parameter as linear:

    paramValue = inValue * (maxValue-minValue) + minValue

What many plugins do is to only expose generic linear parameters with ranges like 0.0 to 1.0 or 0 to 100%, and then do any exp/log mapping internally.

# Changing a parameters value

There are two ways for a host to change a parameter of a plugin.

## UI control

One is the simple `setValue:` family of methods suitable for UI control. Setting a parameter this way will call
the plugins `implementorValueObserver` block.

## Realtime events
The other is by realtime parameter events, which the host schedules using the `audioUnit.scheduleParameterBlock`.
These events does not call the `implementorValueObserver` block, instead they must be handled in the plugins `internalRenderBlock`.

Realtime events allows to use timestamps for precise and jitter-free value changes.

## Implementation

First we make a common realtime-safe function to update values in the DSP state.
Make sure you don't use any Swift code, ARC, Obj-C calls, memory allocation, file I/O, or other unsafe code here, since this will be called from the audio thread!
```
static void MyPluginSetParameter(__unsafe_unretained MyPluginAudioUnit *THIS, AUParameterAddress adr, AUValue val) {
  if(adr < _MyPluginParameterCount)
    DSPValueSet(&THIS->_dspValues[adr], val); // Replace this with whatever code you need to update your DSP state
}
```

After creating your `AUParameterTree`, implement your `implementorValueObserver` and `implementorValueProvider` blocks.

The `implementorValueObserver` block is used to update your DSP state when the host changed a parameter via `setValue:`.
It's *not* called when we receive parameter events in the render block, unlike `tokenByAddingParameterObserver:`.
The block is called on the main thread (on "AUParameterTree.valueAccessQueue").

```
tree.implementorValueObserver = ^(AUParameter *param, AUValue value) {
  MyPluginSetParameter(THIS, param.address, value);
};
```

The `implementorValueProvider` block is called when the AUParameter value needs to be refreshed from your DSP state.
```
tree.implementorValueProvider = ^(AUParameter *param) {
  AUParameterAddress adr = param.address;
  return adr < _MyPluginParameterCount ?
      (AUValue) DSPValueGet(&THIS->_dspValues[adr]) // Replace this with whatever code you need to read your DSP state
    : (AUValue) 0.0;
};
```

In your plugins `internalRenderBlock`, we need to iterate through the realtime events and look for parameter events.
We'll get events here when the host schedules parameter events via the `audioUnit.scheduleParameterBlock`,
but not when the host calls param `setValue:` (in which case `implementorValueObserver` is called instead).

Again, make sure you don't use any Swift code, ARC, Obj-C calls, memory allocation, file I/O, or other unsafe code in your render block!

```
- (AUInternalRenderBlock)internalRenderBlock {
  __unsafe_unretained MyPluginAudioUnit *THIS = self;

  return ^AUAudioUnitStatus(AudioUnitRenderActionFlags *actionFlags, const AudioTimeStamp *timestamp, AVAudioFrameCount frameCount, NSInteger outputBusNumber, AudioBufferList *outputData, const AURenderEvent *realtimeEventListHead, AURenderPullInputBlock pullInputBlock) {

    const AURenderEvent *ev = realtimeEventListHead;
    while (ev) {
      if(ev->head.eventType == AURenderEventParameter
      || ev->head.eventType == AURenderEventParameterRamp) {
        MyPluginSetParameter(THIS, ev->parameter.parameterAddress, ev->parameter.value);
      }
      // NOTE: this is also where you'd handle MIDI events coming from the host
      ev = ev->head.next;
    }
    
    // Do any signal processing and MIDI output here

    return noErr;
  };
}
```

## Timestamping

For jitter-free changes of parameter values, make use of the `ev->head.eventSampleTime` to see where in the current buffer the change should take place.
Timestamping follows the same logic as for MIDI, see [iOS MIDI timestamps](/ios_midi_timestamps).
