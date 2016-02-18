---
layout: default
title: IAA quirks
---

# Using multiple IAA ports

A node app can have several RemoteIO units for multiple outputs or filter ports.

## MACH_RCV_TIMED_OUT bug

When connecting multiple ports, and then disconnecting one of them, an iOS bug (still present in 9.3 beta) makes the still-connected port block the audio thread for several seconds and then the unit stops working, returning MACH_RCV_TIMED_OUT from AudioUnitRender().

To avoid this, we need to restart the unit on disconnection. Once such a unit has been connected, it must stay running until the app has been fully disconnected on all ports. Note that IAA automatically stops the unit on disconnection.

Note that if such an aux unit is running while unconnected, you must mute the buffers in it or it will play through from mic to speaker, causing feedback.

Also note that the same thing applies to the “main” IAA port if the app has one. It must be restarted on disconnection, then check if you should really stop it, and don’t stop if extra IAA ports still connected.

So the rule of thumb is: never stop a RIO unit that was previously connected if there are any other units currently connected.

## Render callback ordering

An app might have multiple remoteIO units, for example one main and one extra for IAA.

The IAA unit might need data produced during our main render callback, so we want to make sure it runs after or main unit.

The IAA units render callback is actually called from its host. So it’s the order of the hosts render callback and our main render callback that matters. This order depends on which render callback started first. So if the host is opened first, and the other app second, the other apps IAA render callback will be called after its main render callback! This is not good, as it introduces one buffer delay: the IAA render callback only has the data produced during the previous main render callback.

One way to fix it might be to check the timestamp and do the main render code from any render callback, when timestamp changed. A problem with that is that a host might render your IAA unit in another buffer size than iOS is rendering your main unit!

Another way that might work is to use render notify callbacks. Will a pre-render notify always happen before *all* render callbacks (including other apps) or only our own? I haven’t investigated this yet.

# Avoiding IAA zombie nodes

The favorite IAA bug that makes node apps unable to load in a host, often showing a message that the user needs to manually launch the app, force-close it, and then try again.

The issue is that if a hosted node is never actually shown in foreground and then disconnected, and then the host process is killed, crashed or thrown out by the iOS task management, then the node doesn’t load the next time.

The solution is to let the node app kill itself when stopping audio if it was not active since last connection.

An IAA node should check if it should stop its audio when backgrounded, and when disconnected from IAA host, and when memberOfActiveAudiobusSession turns false.

```obj-c
// in app delegate

NSTimeInterval appLastActiveTime = 0;

// NOTE: could also probably just use the notification?
- (void)applicationDidBecomeActive:(UIApplication *)application {
    appLastActiveTime = [NSDate timeIntervalSinceReferenceDate];
}

// in audio engine

static NSTimeInterval appLastConnectedTime = 0;
extern NSTimeInterval appLastActiveTime;

- (void)maybeStop {
    if([self shouldKeepRunning]) return;

    if(appLastConnectedTime > appLastActiveTime) {
        NSLog(@"App not active since last connection, maybe cleaning up to avoid IAA zombie process.");

        // Sometimes if the host crashed, the node doesn’t get the disconnect event until woken up.
        // So make sure we’re not being reconnected before exiting!
        // Unfortunately the sleep is needed, so we must keep our audio running while sleeping.

        self.muted = YES;
        AudioOutputUnitStart(mainAudioUnit);
        sleep(1);
        UInt32 connected;
        UInt32 dataSize = sizeof(UInt32);
        AudioUnitGetProperty(mainAudioUnit, kAudioUnitProperty_IsInterAppConnected,
            kAudioUnitScope_Global, 0, &connected, &dataSize);

        if(!connected) {
            NSLog(@"Terminating.");
            AudioOutputUnitStop(mainAudioUnit);
            exit(0);
        }
        NSLog(@"Never mind! Reconnected.");
        self.muted = NO;
    } else {
        AudioOutputUnitStop(mainAudioUnit);
    }
}

- (void)onConnectionChanged {
    if(connected) {
        appLastConnectedTime = [NSDate timeIntervalSinceReferenceDate];
        [self startAudioEngine];
    } else {
        AudioOutputUnitStart(mainAudioUnit); // IAA bug workaround, it will be stopped again in maybeStop if needed.
        [self maybeStop];
    }
}
```
