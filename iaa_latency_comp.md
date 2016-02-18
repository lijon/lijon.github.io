---
title: IAA and Latency Compensation
layout: default
---

# Presentation latency

A host might introduce delay after hosted nodes in the signal chain, for example due to latency compensation or other nodes that adds delay.

It should add the total additional delay at the output + the usual device output latency, to the hostTime passed to `ABLLinkBeatTimeAtHostTime()`. This way, Ableton Link will make sure the output aligns between apps and devices.

When hosting an IAA node that sync to Link, we must let the node app know of our additional delay, also known as presentation latency. It’s the time it takes for the output of the node to reach our hosts output buffer, not including the device output latency.

This is done by adding our additional delay to the mHostTime of the timestamp passed to `AudioUnitRender()` when we render the node.

IMPORTANT: the mHostTime shall never go backwards compared to the last call, or weird stuff will happen. To avoid this, clip the hostTime so that it always increments by at least half a buffer duration until it catches up:

```
AudioTimeStamp t = *inTimeStamp;
t.mHostTime = MAX(
    lastAudioTimestamp + bufferDurationHostTicks/2,
    t.mHostTime + latencyHostTicks
);
lastAudioTimestamp = t.mHostTime;
const AudioTimeStamp *timeStampWithLatency = &t;
```

# Report latency from IAA node to host

For a host to be able to do latency compensation, it must know the introduced latency of its plugins.
Unfortunately IAA does not have a built-in mechanism for this. The Latency property of the RIO unit is not propagated to the IAA node unit in the host.

However, in my [AUM](http://kymatica.com/aum) and [AUFX](http://kymatica.com/aufx) apps, I have implemented this mechanism for reporting latency, through a creative use of remote control events:

## From node

```
void SendLatencyToHost(AudioUnit unit, UInt32 latencyFrames) {
    UInt32 event = (latencyFrames<<8)|0xFF;
    UInt32 dataSize = sizeof(event);
    AudioUnitSetProperty(unit, kAudioOutputUnitProperty_RemoteControlToHost,
        kAudioUnitScope_Global, 0, &event, dataSize);
}
```

The latency is sent when the node becomes connected to host, same time as you would get IAA host icon and HostCallbackInfo, etc.
It is also sent if the latency changes, for example adjusting lookahead time or FFT window size.

If your IAA effect node introduces latency, I recommend you implement the above to get automatic latency compensation in AUM.

## To host

In the host, handle the remote control events like this:

    typedef void (^LatencyReportBlock)(UInt32 latency);

     -(OSStatus) setRemoteControlEventListenerForAudioUnit:(AudioUnit)unit
        latencyReportBlock:(LatencyReportBlock)latencyReportBlock {
        AudioUnitRemoteControlEventListener block = ^(AudioUnitRemoteControlEvent event) {
        dispatch_async(dispatch_get_main_queue(), ^{
            switch (event) {
                case kAudioUnitRemoteControlEvent_TogglePlayPause:
                    self.playing = !_playing;
                    break;
                case kAudioUnitRemoteControlEvent_ToggleRecord:
                    self.recording = !_recording;
                    break;
                case kAudioUnitRemoteControlEvent_Rewind:
                    [self reset];
                    break;
                default:
                    if((event & 0xff) == 0xff) { // got latency report
                        latencyReportBlock(event>>8);
                    }
                    break;
            }
        });
        };
        return AudioUnitSetProperty(unit,
                               kAudioUnitProperty_RemoteControlEventListener,
                               kAudioUnitScope_Global,
                               0,
                               &block,
                               sizeof(block));
    }

You would call `setRemoteControlEventListenerForAudioUnit` on the IAA node unit before initializing it, also passing in a block that is called to report the latency.
