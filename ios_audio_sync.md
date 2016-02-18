---
title: iOS Audio Sync
layout: default
---

# IAA/AU host sync vs Ableton Link sync

Implementing these two are actually quite similar.
The idea is that in each render buffer, you check where you *should be* at the end of this buffer, or actually where you should be at the first sample of the *next* buffer.

## Host sync

Call the host beatAndTempoProc callback that you have cached in a variable.

Take the beat time it returns and add the number of beats that fits in the current buffer according to the tempo you got from the callback.

Note that the tempo might fluctuate and be different for each render cycle, for example if the host is in turn syncing to Link or something else that makes it do slight adjustments for each buffer.

This is expected, I believe it's important that the tempo reported in the callback is the exact tempo for this very buffer. That's how you can know where your beat time will be at the end of the buffer!

You can apply smoothing to the reported IAA host tempo for displaying to the user.

On the other hand, nodes might find it more useful or expect a stable tempo here, and just use the beat time and tempo to "guess" where it should be at the end of the buffer. It would then guess again next buffer, and so on. I'm not 100% sure of what's the best thing to do here, and individual hosts and nodes will most probably handle this differently.

However, even with a stable tempo, nodes must be able to handle some jitter in the beat time. The guessed beatTimeAtEndOfBuffer will not always align with the next beatTimeAtStartOfBuffer (current host beat time).

## Link

Call `ABLLinkBeatTimeAtHostTime()`, passing it the mHostTime given in your render callback timestamp and add the buffer duration in host ticks, as well as the device output latency + any additional delay you’re adding to your audio.

You can calculate the exact tempo for this buffer by checking how many beats that fits in the buffer.

## Advance towards target beat time

Then you need a way to get there in a nice way. It might be enough to just change the playback rate/speed.
More advanced techniques might involve time stretching.

If the jump from your current beat time is too big, or going backwards, then you should handle this by relocating to the correct beat position. The beat time can and will go backwards, when the host rewinds or when Link waits for the phase to reach the next sync quantum.

## Jitter

On most 32-bit devices between `mSampleTime` and `mHostTime` of the timestamp passed to your render callback. Since Link is based on mHostTime, you'll see fluctuations in exact tempo per buffer. If Link is also connected to other Link-enabled apps, the fluctuations will be larger and incorporate adjustments made by Link to keep all peers in sync.

If your app is sensitive to jitter, you might want to smooth the rate of change, but in such a way that no change is lost, only accumulated and spread over a longer time period. For example:

```
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

```
HostCallbackInfo callbackInfo;
callbackInfo.hostUserData = NULL;
UInt32 dataSize = sizeof(HostCallbackInfo);
AudioUnitGetProperty(unit, kAudioUnitProperty_HostCallbacks, kAudioUnitScope_Global, 0, &callbackInfo, &dataSize);
BOOL hostProvidesBeatSync = callbackInfo.hostUserData && callbackInfo.beatAndTempoProc(callbackInfo.hostUserData,NULL,NULL) == noErr;
```

An app should use IAA sync if available, else fall back to Link if enabled.
So the app should use `ABLLinkSetActive()` to deactivate Link while syncing to IAA, and disable or hide the Link button in the app.

