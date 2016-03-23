---
title: iOS Audio Sync
layout: default
---

# Modern sync technologies

Host sync has been available for ages on the desktop, and has been part of the Inter-App Audio technology since it came in iOS 7. It's also how the new Audio Unit extensions in iOS 9 syncs to their hosts. The code examples in this document will show the Inter-App Audio API for host sync, which however is very similar to the new AUv3 API used by Audio Unit extensions.

Ableton Link is a new technology by Ableton, that allows sharing a common tempo and beat grid between apps and devices. It synchronizes beat, phase and tempo but leaves the transport controls (play/stop/rewind) up to each individual app. Each app need to be started and stopped by their own, but will play in sync with the other connected apps once started.

From a users point of view, those two technologies serve quite different purposes. While Ableton Link is great for live jamming, specifically involving multiple users and devices, it lacks a global transport control. There is also no master-slave relationship with Link. Host sync on the other hand, is a master (host) to slave (IAA node app or AU plugin) relationship, and also carries transport state. IAA nodes can also remote-control the transport of the host, and switch to the host app or even [any of the other IAA node apps](/iaa_node_panel.html) hosted by the same host.

Implementing the basic sync mechanism for these two are actually quite similar, however.
The idea is that in each render buffer, you check where you *should be* at the end of this buffer, or actually where you should be at the first sample of the *next* buffer.

# Ableton Link

At the top of your render callback, you call `ABLLinkBeatTimeAtHostTime()`, passing it the mHostTime that was passed in your render callbacks timestamp argument. But first, you must add the buffer duration in host ticks as well as the device output latency, plus any additional delay you’re adding to your audio.

This way, you get the precise beat time for the actual time corresponding to the end of the buffer (or actually the start of the next buffer).

The beat time will not advance in exact equal increments, even when the tempo is fixed. This is because of small adjustments Link does to keep all the peers in sync.

If you need to, you can calculate the exact tempo for the current buffer by checking how many beats that fits in the buffer, using the `ABLLinkBpmInRange()` function from ABLLinkUtils.h.

Link also provides functions to get and set the session tempo, setting the sync quantum, and more. See [ableton.github.io/linkkit](http://ableton.github.io/linkkit) for full API documentation.

# Host sync

Host sync is done through a set of callbacks contained in a HostCallbackInfo struct, that you retrieve from the host when connected and cache in a variable that is safe to access from the audio thread.

```objc
HostCallbackInfo hostCallbackInfo = {0,};

UInt32 size = sizeof(HostCallbackInfo);
AudioUnitGetProperty(_audioUnit,
    kAudioUnitProperty_HostCallbacks,
    kAudioUnitScope_Global, 0,
    &_hostCallbackInfo, &size);
```

The two important callbacks in the HostCallbackInfo struct are `beatAndTempoProc` and `transportStateProc2`.

## Beat time clock

At the start of each render cycle, call the `beatAndTempoProc()` callback. You must call this in the render callback (audio thread), not the main thread!

```objc
Float64 iaa_tempo = 0, iaa_beat = 0;

OSResult result = _hostCallBackInfo.beatAndTempoProc(
    _hostCallBackInfo.hostUserData,
    &iaa_beat,
    &iaa_tempo);

if(result == noErr) {
    // use given tempo and beat here
}
```

Take the beat time it gives and add the number of beats that fits in the current buffer according to the tempo you got from the callback. This will give you the expected beat time at the end of the buffer.

The given tempo should be used as an indication of where the beat time will be at the end of the buffer, there's no guarantee that this will be precisely correct. At the next render cycle, you know where you are and again you guess where you should be at the end of the buffer.

Note that the tempo might fluctuate and be different for each render cycle, or it might be stable and smoothed. The fluctuations might come because the host is in turn syncing to something else, like Ableton Link or MIDI clock.

My mixer and host app [AUM](http://kymatica.com/aum) as of version 1.0 sends a fluctuating tempo that is the precise tempo for the current buffer, but since many IAA node apps might not be prepared for this it will instead send the stable session tempo beginning with version 1.1.

However, even with a stable tempo, nodes must be able to handle jitter in the beat time. The guessed beat time for end of buffer will not always align with the reported beat time you get during the next render cycle. More on this later in this document.

## Transport state

Host sync has transport state that lets the node/plugin know if the transport is playing and if it's recording, as well as some more esoteric information that we won't focus on here.

To get the current transport state, you use the corresponding callback in the same way as the beat and tempo callback:

Call the `transportStateProc2()` function at the top of your render callback, and detect if the play state has changed. You pass NULL for any values that you are not interested in. Start playing your audio *in the same buffer as the host changes state to playing*.

```objc
if (_hostCallbackInfo.hostUserData) {
    Boolean isPlaying = NO;

    OSStatus result = _hostCallbackInfo.transportStateProc2(
        _hostCallbackInfo.hostUserData,
        &isPlaying,
        NULL, NULL, NULL, NULL, NULL, NULL);

    if (result == noErr && isPlaying != _wasPlaying) {
        _wasPlaying = isPlaying;
        // do what you need here to start or stop your things
    }
}
```

## Starting at negative beat time

When the buffer comes where state changes to playing, the beat time for the start of the buffer might very well be negative! This could happen if the host has pre-roll, or if it's syncing to Link and waiting for sync quantum phase. In those situations, the exact beat time 0 will not be at the start of the buffer, but somewhere in the middle of it.

A node or plugin must be careful and calculate the frame offset where the first beat actually is within the buffer.

There might also be multiple buffers with negative beat time before the buffer containing the start beat frame comes. So, it's important that a node must treat negative beat time to mean "just sit and wait until time 0".

## Transport panel

It's nice for an IAA node app to display a transport control panel and the current time for its host. To get the current state, just use the `transportStateProc2` again, now passing in some more variables that we are interested in:

```objc
if (_hostCallbackInfo.hostUserData) {
    Boolean isPlaying = NO;
    Boolean isRecording = NO;
    Float64 outCurrentSampleInTimeLine = 0;

    OSStatus result = _hostCallbackInfo.transportStateProc2(
        _hostCallbackInfo.hostUserData,
        &isPlaying,
        &isRecording,
        NULL,
        &outCurrentSampleInTimeLine,
        NULL, NULL, NULL);

    if (result == noErr) {
        // update your UI here
    }
}
```

Note that the current sample time is given in the [host sample rate](/iaa_sample_rates.html), so make sure to use that when converting it to seconds.

You can call the above function periodically to update your UI, or if you are already calling it in your audio thread you can store the variables from there and read them in an UI timer to update your transport panel.

Note that this use here is only for displaying the transport state and time location for the user, responding to the actual `playing` state should be done in your audio thread.

For remote controlling the host transport, you send AudioUnitRemoteControlEvents:

```objc
- (void)sendEventToRemoteHost:(AudioUnitRemoteControlEvent)state {
    if(_outputUnit) {
        UInt32 controlEvent = state;
        UInt32 dataSize = sizeof(controlEvent);
        AudioUnitSetProperty(_outputUnit, 
            kAudioOutputUnitProperty_RemoteControlToHost, 
            kAudioUnitScope_Global, 0,
            &controlEvent, dataSize);
    }
}

- (void)rewindTapped {
    [self sendEventToRemoteHost:kAudioUnitRemoteControlEvent_Rewind];
}

- (void)playTapped {
    [self sendEventToRemoteHost:kAudioUnitRemoteControlEvent_TogglePlayPause];
}

- (void)recTapped {
    [self sendEventToRemoteHost:kAudioUnitRemoteControlEvent_ToggleRecord];
}
```

# Towards target beat time...

Now that you know the beat time at the end of the current buffer, you need a way to get there in a nice way.

It might be enough to just change the playback rate/speed.
More advanced techniques might involve time stretching.
If your app is trigger-based, for example a drum machine, just calculate the frame offsets for each event to see where in the buffer they should start playing.

If the jump from your current beat time is too big, or going backwards, then you should handle this by relocating/skipping to the correct beat position. The beat time can and will go backwards, when the host rewinds or when Link waits for the phase to reach the next sync quantum.

## Jitter

On most 32-bit devices there's jitter between `mSampleTime` and `mHostTime` of the timestamp passed to your render callback. Since Ableton Link is based on `mHostTime`, you'll see fluctuations in the incrementations of the beat time, and thus also in the calculated precise tempo for each buffer. If Link is also connected to other Link-enabled apps, the fluctuations might be larger and incorporate adjustments made by Link to keep all peers in sync.

One question I had regarding Link: Does the jittery beat time average out in the long run to stay in sync with a theoretical ideal clock source? Ableton responded that yes, this should be true but with the caveat that the hosttime clock of one of the devices in a session will be used as reference. Clocks can have slightly different frequency from each other, so it's not true that it will match up with a theoretical "ideal" clock - it will match up with the actual physical clock of one of the devices on the network and the others will make slight adjustments to stay in sync with that.

If your app is sensitive to jitter, you might want to smooth the rate of change, but in such a way that no change is lost, only accumulated and spread over a longer time period. For example something like this:

```objc
Float64 rate = stableTempo; // ideal rate
Float64 syncRate = actualTempo; // synchronized rate
// rateRemain is an accumulator for change not yet used
Float64 diff = syncRate-rate + rateRemain;
Float64 alpha = (Float64)frames/5120.0;
rate += diff * alpha;
rateRemain = diff * (1.0-alpha);
// now use rate here
```

# Host sync or Link, or both?

I think it makes a lot of sense for an IAA app to support both host sync and Ableton Link. For an AU extension, host sync is the only reasonable option.

In my opinion, an app should use IAA sync if available, else fall back to Link if enabled. This is because a user expects a hosted node app to sync with the host. So the app should use `ABLLinkSetActive()` to deactivate Link while syncing to IAA, and disable or hide the Link button in the app. You can also provide an "IAA host sync" toggle just in case the user wants to use Link instead and are not hosting the app inside a Link-enabled host such as [AUM](http://kymatica.com/aum).

## Detection of IAA sync

To detect if host provides IAA sync or not, use the following code. You could call this directly after being connected to host, on the main thread after you've read and cached the HostCallbackInfo to a variable:

```objc
BOOL hostProvidesBeatSync = _hostCallbackInfo.hostUserData &&
    _hostCallbackInfo.beatAndTempoProc(
        _hostCallbackInfo.hostUserData, NULL, NULL) == noErr;
```
