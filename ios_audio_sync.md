---
title: iOS Audio Sync
layout: default
---

# IAA/AU host sync vs Ableton Link sync

Implementing these two are actually quite similar.
The idea is that in each render buffer, you check where you *should be* at the end of this buffer, or actually where you should be at the first sample of the *next* buffer.

## Host sync

At the start of each render cycle, call the host beatAndTempoProc callback that you have cached in a variable. You must call this in the render callback (audio thread), not the main thread!

Take the beat time it returns and add the number of beats that fits in the current buffer according to the tempo you got from the callback.

The given tempo should just be used as an indication of where the beatTime will be at the end of the buffer, there's no guarantee that this will be precisely correct.

At the next render cycle, you know where you are and again you guess where you should be at the end of the buffer.

Note that the tempo might fluctuate and be different for each render cycle, or it might be stable and smoothed. The fluctuations might come because the host is in turn syncing to something else, like Ableton Link or MIDI clock.

You can apply smoothing to the reported IAA host tempo if you need to assure a non-fluctuating tempo.

My mixer and host app [AUM](http://kymatica.com/aum) currently sends a fluctuating tempo that is the precise tempo for the current buffer. But this will probably change in the next update, since many IAA node apps might not be ready for this.

However, even with a stable tempo, nodes must be able to handle jitter in the beat time. The guessed beat time for end of buffer will not always align with the reported beat time you get during the next render cycle.

## Link

With [Ableton Link](http://ableton.github.io/linkkit/), you call `ABLLinkBeatTimeAtHostTime()`, passing it the mHostTime given in your render callback timestamp and add the buffer duration in host ticks, as well as the device output latency + any additional delay you’re adding to your audio.

This way, you already get the precise beat time for the end of the buffer (or actually the start of the next buffer).

Also with link, the beat time will not advance in exact equal increments, even when the tempo is fixed. This is because of small adjustments Link does to keep all the peers in sync.

You can calculate the exact tempo for this buffer by checking how many beats that fits in the buffer.

## Transport state

Host sync also has transport state. It lets the node/plugin know if the transport is playing and if it's recording. You should use this in the same way as the beat and tempo callback: Call the `transportStateProc2()` function at the top of your render callback, and detect if the play state has changed. Start playing in the same buffer as the host changes state to playing.

NOTE: When the buffer comes where state changes to playing, the beat time for the start of the buffer might very well be negative! This could happen if the host has pre-roll, or if it's syncing to Link and waiting for sync quantum phase. In those situations, the exact beat time 0 will not be at the start of the buffer, but somewhere in the middle of it. A node or plugin must be careful and calculate the frame offset where the first beat actually should start within the buffer.

Ableton Link has no transport state, each app controls their start/stop individually.

## Advance towards target beat time

Now you know the beat time, and you need a way to get there in a nice way. It might be enough to just change the playback rate/speed.
More advanced techniques might involve time stretching.

If the jump from your current beat time is too big, or going backwards, then you should handle this by relocating to the correct beat position. The beat time can and will go backwards, when the host rewinds or when Link waits for the phase to reach the next sync quantum.

## Jitter

On most 32-bit devices between `mSampleTime` and `mHostTime` of the timestamp passed to your render callback. Since Link is based on `mHostTime`, you'll see fluctuations in the incrementations of the beat time, and thus also in the calculated precise tempo for each buffer. If Link is also connected to other Link-enabled apps, the fluctuations will be larger and incorporate adjustments made by Link to keep all peers in sync.

If your app is sensitive to jitter, you might want to smooth the rate of change, but in such a way that no change is lost, only accumulated and spread over a longer time period. For example:

```objc
Float64 rate = stableTempo; // ideal rate
Float64 syncRate = actualTempo; // synchronized rate
// rateRemain is an accumulator for change not yet used
Float64 diff = syncRate-rate + rateRemain;
Float64 alpha = (Float64)frames/5120.0; // filter coeff depends on buffer size
rate += diff * alpha;
rateRemain = diff * (1.0-alpha);
// now use rate here
```

## Detect if host provides IAA sync

In my opinion, an app should use IAA sync if available, else fall back to Link if enabled. This is because a user expects a hosted node app to sync with the host. So the app should use `ABLLinkSetActive()` to deactivate Link while syncing to IAA, and disable or hide the Link button in the app.

To detect if host provides IAA sync or not, use the following code. You could call this directly after being connected to host, on the main thread, at the same time as you anyway cache the host callbackInfo to a variable for use in the render callback.

```objc
HostCallbackInfo callbackInfo; // global or instance variable

callbackInfo.hostUserData = NULL;
UInt32 dataSize = sizeof(HostCallbackInfo);

AudioUnitGetProperty(unit,
    kAudioUnitProperty_HostCallbacks,
    kAudioUnitScope_Global, 0,
    &callbackInfo, &dataSize);

BOOL hostProvidesBeatSync = callbackInfo.hostUserData
    && callbackInfo.beatAndTempoProc(callbackInfo.hostUserData,
        NULL, NULL) == noErr;
```


