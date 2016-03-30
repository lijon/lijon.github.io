---
layout: default
title: IAA quirks
---

# Using multiple IAA ports

A node app can have several RemoteIO units for multiple outputs or filter ports.

With multiple ports, I mean actually having individual units, so that your app can output simultaneously on each port. Then you can simply check which unit is passed to the IsInterAppConnected callback to know which unit got (dis)connected.

If you're instead using a single RemoteIO unit for multiple AudioComponentDescriptions, you can get the ACD like this to see which one was used to connect:

```objc
AudioComponentDescription acd;
UInt32 dataSize = sizeof(acd);
AudioUnitGetProperty(inUnit, 
    kAudioOutputUnitProperty_NodeComponentDescription,        
    kAudioUnitScope_Global, 0, &acd, &dataSize);
```

## MACH_RCV_TIMED_OUT bug

When connecting multiple ports, and then disconnecting one of them, an iOS bug (still present in 9.3 beta) makes the still-connected port block the audio thread for several seconds and then the unit stops working, returning MACH_RCV_TIMED_OUT from AudioUnitRender().

To avoid this, we need to restart the unit on disconnection. Note that IAA automatically stops the unit on disconnection, so we must explicitly start it again.

Once such a unit has been connected, it must stay running until the app has been fully disconnected on all ports.

So at disconnection, we can always restart the unit, and *then* check if we should stop our audio engine for real, if *all* our ports are disconnected and we're in the background and shouldn't keep playing, etc.

```objc
// FIXME: example code using IsInterAppConnected callback,
// also setting a mute flag..
```

IMPORTANT: If such an aux unit is running while unconnected, you must mute the buffers in it or it will play through from mic to speaker, causing feedback.

Also note that the same thing applies to the “main” IAA port if the app has one. It must be restarted on disconnection, then check if you should really stop it, and don’t stop if extra IAA ports still connected.

So the rule of thumb is: *Restart a RIO unit on disconnection, and never stop a RIO unit that was previously connected if there are any other units currently connected*.

## Render callback ordering

An app might have multiple remoteIO units, for example one main and one extra for IAA.

The IAA unit might need data produced during our main render callback, so we want to make sure it runs after or main unit.

The IAA units render callback is actually called from its host. So it’s the order of the hosts render callback and our main render callback that matters. This order depends on which render callback started first. So if the host is opened first, and the other app second, the other apps IAA render callback will be called before its main render callback! This is not good, as it introduces one buffer delay: the IAA render callback only has the data produced during the previous main render callback.

One way to fix it is to check the timestamp and do the main render code from any render callback, when the timestamp changed. A problem with this approach is however that it assumes all units will be rendered with equal buffer sizes. A host could potentially render your IAA unit in another buffer size than iOS is rendering your main unit, for example splitting the cycle into smaller buffers.

```obj-c
static void mainCallback(void* inRefCon,
    UInt32 inNumberFrames,
    const AudioTimeStamp* inTimeStamp,
    AudioBufferList* ioData)
{
    // do the main render routine as soon as the required number of
    // frames have elapsed, this handles AUs being called from
    // different hosts, while still only calling the main render
    // routine once for each buffer
    static Float64 lastSampleTime = 0;
    if (inTimeStamp->mSampleTime - lastSampleTime < inNumberFrames) {
        return;
    }
    lastSampleTime = inTimeStamp->mSampleTime;
    // produce/process audio here and put it in some internal buffers.
    // here we also pull audio from hardware input or IAA filter ports
    // using AudioUnitRender();
}

// The actual render callback for any RemoteIO unit
static OSStatus renderCallback(void* inRefCon,
    AudioUnitRenderActionFlags* ioActionFlags,
    const AudioTimeStamp* inTimeStamp,
    UInt32 inBusNumber,
    UInt32 inNumberFrames,
    AudioBufferList* ioData)
{
    mainCallback(inRefCon, inNumberFrames, inTimeStamp);
    // copy audio from our internal buffers to ioData here.
}
```

The problem with this is that a host could very well change the system timestamps before passing them to the node when rendering them! For example, AUM adjust the mHostTime to make Link-enabled apps sync even though AUM is doing latency compensation, but it doesn't change mSampleTime.

So in summary, there's still no good solution to this problem.

# Avoiding IAA zombie nodes

The favorite IAA bug that makes node apps unable to load in a host, often showing a message that the user needs to manually launch the app, force-close it, and then try again.

The issue is that if a hosted node is never actually shown in foreground and then disconnected, and then the host process is killed, crashed or thrown out by the iOS task management, then the node doesn’t load the next time.

According to my tests, the node will fail to load if it hasn't been in the foreground *since last connection*.

The solution is to let the node app kill itself when stopping audio if it was not active since last connection.

By terminating itself, the app makes sure that it will not be running without being visible in the multi-task view. If the app becomes unloadable after being in foreground, user can at least easily see the app and swipe it out, which is a lot better than having an invisible process running. Audiobus 2 handles this simply by forcing the node app to come to foreground by switching to it after connection, which can be quite annoying when loading many apps at once. (And IAA by itself has no standard for switching back to host, even if it would be possible for the node to open the host URL when coming foreground after becoming connected to host.)

```objc
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

        // Sometimes if the host crashed, the node doesn’t get
        // the disconnect event until woken up.
        // So make sure we’re not being reconnected before exiting!
        // Unfortunately the sleep is needed, so we must keep our
        // audio running while sleeping.

        self.muted = YES;
        AudioOutputUnitStart(mainAudioUnit);
        sleep(1);
        UInt32 connected;
        UInt32 dataSize = sizeof(UInt32);
        AudioUnitGetProperty(mainAudioUnit,
            kAudioUnitProperty_IsInterAppConnected,
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
        // IAA bug workaround: restart unit here.
        // It will be stopped again in maybeStop if needed.
        AudioOutputUnitStart(mainAudioUnit);
        [self maybeStop];
    }
}
```

An IAA node should check if it should stop its audio, by calling `maybeStop`, when backgrounded, when disconnected from IAA host, and when `memberOfActiveAudiobusSession` turns false if you're using Audiobus.

Note that this workaround is not only to avoid zombies, but also to avoid invisible IAA node apps running in the background, possibly eating CPU or even producing audio without the user knowing which app is doing it, and having no way to see it in the multi task view to terminate the app.

## Cleanup from host side
Another important thing is that the host uninitialize and dispose its hosted IAA node units when done with them.

What might not be obvious is that a host should also do this when it terminates, if there are any nodes connected at that point. This happens for example if the user swipes out the host app from the multi-task view. Use the `applicationWillTerminate:` AppDelegate method!