---
title: IAA and sample rates
layout: default
---

# Host sample rate

The RIO unit of the node has a client side (for the node) and the “outer" side (host, or hardware if not connected to host)

​A host must run nodes in the hardware sample rate, and it sets the format on its client side of the IAA node unit, which equals the outer side of the RIO unit from the node’s perspective.

The node can detect this and follow the host sample rate, or it could ignore it (as most IAA node apps does, currently). When ignored, the RIO will do sample rate conversion.

IAA host gives current time in samples. Since node and host might run at different sample rates, we must use the host SR for converting to seconds.

```objc
static void UpdateHostSampleRate(AudioUnit unit) {
    AudioStreamBasicDescription asbd = {0,};
    UInt32 dataSize = sizeof(asbd);
    AudioUnitGetProperty(unit,
        kAudioUnitProperty_StreamFormat,
        kAudioUnitScope_Output, 0, &asbd, &dataSize);
    hostSampleRate = asbd.mSampleRate;
}
```

## Detect SR changes

The host might change SR after connection. When an IAA host changes sample rate, it must uninitialize all the hosted nodes, change their stream format to use the new SR, and then initialize it again. The IAA node sees this as a disconnect-reconnect, at least most of the times.

NOTE: Using the AVAudioSession route-change notification to update the sample rate *does not* work reliable. It works when plugging headphones and audio interfaces in/out, but most often not when the host changes sample rate. The technique used here catches all situations, also when plugging hardware.

So we should use a property listener on this property:

```objc
static void StreamFormatCallback(void *inRefCon,
    AudioUnit inUnit, AudioUnitPropertyID inID,
    AudioUnitScope inScope, AudioUnitElement inElement)
{
    if(inScope == kAudioUnitScope_Output && inElement == 0) {
        UpdateHostSampleRate(inUnit);
    }
}
```

And add the property listener when setting up IAA for our main audio unit:

```objc
AudioUnitAddPropertyListener(unit, 
    kAudioUnitProperty_StreamFormat,
    StreamFormatCallback, (__bridge void * _Nullable)(self));
```

You would also call `UpdateHostSampleRate(inUnit)` when connected, a good time is when getting the other data such as HostCallbackInfo and host icon.

## Flexible sample rate

Running the node in a different SR than the host is sub-optimal, since it introduces sample rate conversion and odd buffer sizes that changes for each render cycle. It’s better to adjust your sample rate to follow the host, by setting the stream format on both InputScope 0 (for output to host) and OutputScope 1 (for input from host, if the node is an effect) to use the same sample rate, and make sure your DSP code handles the new rate.

This is especially important for IAA effects: The sample rate conversion lead to unexpected differences between input and output buffer sizes (maybe because of an iOS bug), making it impossible to implement a straight signal chain without having to use ringbuffers.

Note that listening on `kAudioUnitProperty_StreamFormat` (kAudioUnitScope_Output, element 0) covers both the case when it’s connected to host and when it’s running standalone directly to hardware. So we can use this same mechanism to know the current sample rate on the “outer” side, and if we want to follow it: set our own client-side sample rate to match it.

