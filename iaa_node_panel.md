---
layout: default
title: IAA node panel
---

Some have been wondering how I did the IAA node panel UI in my [AUFX apps](http://kymatica.com/aufx), shown when tapping the host icon while connected to a host.

Instead of just switching to the host, it shows a popup with icons for all nodes currently hosted in the same host, allowing easy switching between them.

Even though I've found no API to get such a list of sibling IAA nodes, there's a public built-in UIView subclass in `CoreAudioKit` that shows this panel: `CAInterAppAudioSwitcherView`

It's that simple. Just create an instance of this view, set the corresponding audio unit, and add it to some container view:

```objc
CAInterAppAudioSwitcherView *v = 
    [[CAInterAppAudioSwitcherView alloc] 
        initWithFrame:CGRectMake(0, 0, width, 80)];
[v setOutputAudioUnit:myAudioEngine.audioUnit];
[containerView addSubview:v];
```
