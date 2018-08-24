---
title: iOS MIDI timestamps
layout: default
---

# The importance of timestamping

It's important to make use of the timestamping when receiving and sending MIDI, otherwise you'll get ugly timing jitter that ruins the music.

# AUv3 plugins

AUv3 plugins allows the use of timestamping in sample time for each render cycle, allowing sample perfect timing.

## Receiving MIDI from an AUv3

AUv3 plugins have the possibility to send MIDI to the host, via an `MIDIOutputEventBlock` property.

As the headers say, the host can set this block and the plug-in can call it in its renderBlock to provide to the host the MIDI data associated with *the current render cycle*.

The timestamp you get from an AUv3 in the `MIDIOutputEventBlock` should be an absolute sample time just like the mSampleTime passed in to your main render callback. Plugins not following this convention won't work correctly!

You might rather want to know at which frame in the current buffer the event is timestamped. To convert an absolute sample tame to frame offset within the current buffer, you must subtract the `mSampleTime` associated with the current render cycle from it. In your main render callback, save the `inTimeStamp->mSampleTime` somewhere, and use it when converting the timestamp in your `MIDIOutputEventBlock`. Also make sure to clip it in case the AUv3 is sending events with invalid timestamps, outside the bounds of the current cycle. So save `inNumberFrames` as well for this purpose.

In your render callback:

```
THIS->_currentTimeStamp = *inTimeStamp;
THIS->_currentBufferSize = inNumberFrames;
```

In the `MIDIOutputEventBlock`:

```
AUEventSampleTime now = 
    (AUEventSampleTime)_currentTimeStamp.mSampleTime;
UInt32 offsetFrames =
    (UInt32)(eventSampleTime >= now ? eventSampleTime-now : 0);
if(offsetFrames >= _currentBufferSize)
    offsetFrames = 0;
```

The `offsetFrames` should then be stored together with the event for later use.

Note that the `MIDIOutputEventBlock` will be called from the audio thread just like your render callback, so follow the rules of realtime safety! (No obj-c, swift, blocking, locks, memory allocations or file I/O).

## Sending MIDI to an AUv3

Then, when sending MIDI to an AUv3, you use this relative frame offset (which is in the range 0 to `_currentBufferSize`) and add
`AUEventSampleTimeImmediate` to it:

```
safeSelf->_midiScheduleBlock(AUEventSampleTimeImmediate
    + offsetFrames, cable, length, data);
```

If you rather would store the original events absolute sample time, and subtract the current sample time when sending to an AUv3, that is absolutely fine as well.

As the headers say about the timestamp in `AUScheduleMIDIEventBlock`: "The sample time (timestamp->mSampleTime) at which the MIDI event is to occur. When scheduling events during the render cycle (e.g. via a render observer) this time can be AUEventSampleTimeImmediate plus an optional buffer offset, in which case the event is scheduled at that position
in the current render cycle.". However, as far as my tests have shown, it does *not* work to send absolute sample time in this case, even if the documentation implies that. So just use `AUScheduleMIDIEventBlock + offsetFrames` and make sure to schedule events from the audio thread, otherwise the timestamp can't be associated with the current render cycle.

# Inter-App Audio nodes

IAA also allows sample time (frame offset) timestamps when sending MIDI to an IAA node, allowing sample perfect timing. Not all IAA synths does this, however, so make sure to test with one known to do it right.

Unfortunately IAA has no concept of MIDI outputs, so receiving MIDI from an IAA app needs to be done via CoreMIDI virtual endpoints.

## Receving MIDI in the IAA node

To receive MIDI in your IAA node app, hook up the MIDI reception callbacks:

```
AudioOutputUnitMIDICallbacks cb;
cb.userData = (__bridge void * _Nullable)(myUserData);
cb.MIDIEventProc = IAAMIDICallback;
cb.MIDISysExProc = IAASysExCallback;
AudioUnitSetProperty (mainAudioUnit,
    kAudioOutputUnitProperty_MIDICallbacks,
    kAudioUnitScope_Global,
    0,
    &cb,
    sizeof(cb));
```

In the callback, you'll be passed the frame offset for this event. 

```
static void IAAMIDICallback(void *userData, UInt32 status, UInt32 inData1, UInt32 inData2, UInt32 inOffsetSampleFrame) {
    // do stuff here
}
```

Use the frame offset to produce sound at the correct samples in the current buffer cycle.

The SysEx callback has no timestamping, so we'll just ignore that for now.

Note that these callbacks will be called from the audio thread, so follow the rules of realtime safety! (No obj-c, swift, blocking, locks, memory allocations or file I/O).

## Sending MIDI to an IAA node

When sending MIDI to an IAA node, pass the frame offset for the event in the call to `MusicDeviceMIDIEvent`. Unfortunately this function is fixed to 3 byte messages, so you might need some logic to find out the correct size of the event you're going to send. Also here, we clip the offset to keep it within the bounds of the current buffer size.

```
MyEvent *ev = ... // this is some struct holding your event
UInt32 t = (UInt32)MIN(_currentBufferSize,MAX(0,ev->frameOffset));
if(ev->length > 0 && ev->data[0] == 0xf0)
    MusicDeviceSysEx(weakSelf->_audioUnit, ev->data, ev->length);
else if(ev->length==3)
    MusicDeviceMIDIEvent(weakSelf->_audioUnit, ev->data[0], ev->data[1], ev->data[2], t);
else if(ev->length==2)
    MusicDeviceMIDIEvent(weakSelf->_audioUnit, ev->data[0], ev->data[1], 0, t);
else if(ev->length==1)
    MusicDeviceMIDIEvent(weakSelf->_audioUnit, ev->data[0], 0, 0, t);
```

Note that you should call this from your audio thread, otherwise the timestamp can't be associated with the current render cycle.

# CoreMIDI

CoreMIDI uses host ticks instead of sample time. Converting between them can be really messy, especially since the CoreMIDI scheduler delivers the events "on time", which is already too late if one wants to act upon the event in the current render cycle!

For virtual inputs, you can set `kMIDIPropertyAdvanceScheduleTimeMuSec` to a non-zero value to get events ahead of time, at the moment they are sent. However, it only works on virtual inputs, so it can't be used when listening directly on CoreMIDI sources such as virtual outputs from other apps or hardware MIDI inputs. Also, since multiple sources could send to the same virtual input, you can't guarantee that the events are received in chronological order. Thus, you need a realtime-safe priority queue to store and sort the events, etc.

# Bonus

Sometimes you might receive multiple MIDI events in the same data array, and you need to split them up into separate events. For example, AUv3 midi output can contain multiple events in the same byte buffer. Here's a simple and fast function to do just that:

```
void SplitMIDIEvents(MyMIDISource *src, MyMIDIPacket *ev) {
    if(ev->length==0) return;
    const UInt8 *data = ev->data;
    if(*data == 0xF0) {
        src->_recvfuncptr(src, ev);
        return;
    }
    const UInt8 *end = &ev->data[ev->length];
    const UInt8 *p = data+1;
    while(1) {
        if((*p & 0x80) || p==end) {
            AUMMIDIPacket ev2 = {
                .timestamp = ev->timestamp,
                .length = p-data,
                .data = data,
            };
            src->_recvfuncptr(src, &ev2);
            if(p==end) break;
            data = p;
        }
        p++;
    }
}
```

Where MyMIDISource is a struct or object containing a function pointer (_recvfuncptr) for handle individual MIDI events, and MyMIDIPacket is a struct that simply contains a timestamp, a length and a byte array for MIDI data.
