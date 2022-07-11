---
title: 'Prototyping a stationary bike "step" counter with Swift Charts'
type: posts
tags: ['swiftui', 'swift charts', 'coremotion', 'prototyping']
categories: ['Development']
description: 'Prototyping a stationary bike "step" counter with Swift Charts'
---

## Prototyping a stationary bike "step" counter with Swift Charts
{{< datecodeanchorslug date="July 11th, 2022" >}}

I was recently chatting with a friend about SwiftUI, and I admitted that I had skipped the awkward first year entirely, but since I had gotten into it, it was reigniting my joy of the platform. "Why?" he asked. I had to think for a moment. "I love prototyping, and SwiftUI removed blockers that I had stopped noticing."

Back in the age of the dinosaurs, when I was doing a technical proof of concept, I preferred to write Objective-C UIs in code. There's something very consonant, very nimble, about being able to take in an entire logical graph in a single zone of attention: maybe just one file you can see all of. When you're just finding out if something is possible, the tradeoffs of that approach are compensated in [flow state](https://en.wikipedia.org/wiki/Flow_(psychology)).

SwiftUI feels the same but with more concision, and without the tradeoffs.

In the present, with a particular itch to scratch: I have a cheap stationary bike, perfect for my workouts in every respect **except** its [inhumane](https://dl.acm.org/doi/10.5555/333103) display interface. And I spend so much time looking at the display and thinking about it! Should I run its signal wires into an MCU and make my own display? Should I use computer vision to consume its data? Is there something like [Van Eck phreaking](https://en.wikipedia.org/wiki/Van_Eck_phreaking) for LEDs? 

Or, can I forget about the display entirely, estimate the bike's stationary km/h using sensors from my phone when it's in the bike's reading material holder, and calculate everything else that's interesting? 

### Can I?

I know from working with a [9-DoF](https://embeddedcomputing.com/technology/analog-and-power/basics-of-6dof-and-9dof-sensor-fusion) sensor in hardware projects that the odds are decent there's some perceptible movement to use. But while I have theories about which sensors are likely to register the oscillation of a stationary bike (guesses: accelerometer and gyroscope), I wouldn't assume: it's even possible that pedaling the bike produces a field that could be picked up reliably on another sensor. No doubt any data will be small and noisy. I guess I need a prototype!

The iPhone's 9-DoF sensors output readings which are arrays of doubles, but like all samples of electromechanical devices, such as the [MEMs](https://en.wikipedia.org/wiki/Microelectromechanical_systems) sensors in an iPhone, these samples are just pointillist representations of real waves. When we talk about extracting meaning from the samples, we use perceptual language like smoothing. If this works, we will most easily see it visually, so we should [**chart**](https://developer.apple.com/documentation/Charts) the sensors.

### Swift Charts

When I first decided to do this post, I checked out all the chart packages for SwiftUI, but none of them quite spoke to me. I put the idea aside for a few days and decided to come back to it later and just pick one, and when I came back to it, WWDC had happened and Apple had released a really snazzy chart framework just for me, thanks Apple!{{< footnote "1" >}}(German has a highly-pejorative word for taking action purely for the sake of taking action, but it lacks a word for reaping the benefits of inaction.){{< /footnote >}} This felt like more supporting data for my warm Swift prototyping feelings. **It also means you can only run this code from an Xcode 14 beta or later to a real device running an iOS 16 beta or later.**

### Goals:

Write very minimal code for a single view with a chart of every sensor. Don't extensively state- and error-handle; it should be run in portrait mode and it's fine to assume a functioning device. Don't worry about nice user interactions{{< footnote "2" >}}(I think there is a difference between a UI and a DI or developer interface, and like a wireframe and a design artwork, it can be a pretty good idea to prevent them from being confused with each other){{< /footnote >}}. Make it possible to:

1. see all the sensors, 
2. turn off the views of the sensors without relevant data, 
3. break out the three sensor axes to inspect them, and, for promising sensors,
4. be able to smooth the wave and count the number of wave peaks (which are equivalent to pedaling "strides").

The technical prototype is the question "is this possible?"; what's inside it is the least complexity that would provide the answer. A product would present a second question, which is "can this be generalized?", and the prudent answer is "not necessarily"{{< footnote "3" >}}(although I have some theories){{< /footnote >}}.

In an app, I wouldn't have a model and an interface in the same file. However, another thing I like about SwiftUI is the ability to give someone a single file they can drop into an app and try out just by initializing it in the App struct, with a UI and everything, wow. Trying things out is the name of the game here, so a single file will be my preference in this blog. But I added some fancy comment formatting so it's easy to draw attention to interesting things from inside the file, and indicate when we're in different kinds of modules.

## ContentView.swift {#thecode}

([view on Github](https://github.com/Halle/StationaryBikeStepCounter/blob/main/ContentView.swift)) 

{{< highlighter color="yellow" ranges="13-13, 37-38, 56-56, 79-79, 121-126, 161-161, 232-235">}}
{{<  includegithubraw "StationaryBikeStepCounter/contents/ContentView.swift" >}}
{{< /highlighter >}}



Here is a video of my using the completed prototype. It works; you can see that I turn off the sensors which aren't reacting to my pedaling at all, then check the three sensors which do react. I turn off the first two because I don't think the waveform of the bike oscillation is very clear. But in the last one, I can see it quite clearly on axis z. So, I turn on the low-pass filter, turn it up almost all the way, and set the quantizing to a very large number. It accurately counts how often I pedal per window.

{{< video src="/videos/SensorCharts.MP4" type="video/mp4" preload="auto" >}}

## Extro

✂ - - - *Non-code thoughts, please stop reading here for the code-only experience* - - - ✂

I haven't been a fan of what I would describe as the Corona Aesthetic in independent filmmaking (2-4 people deal with something ambiguous and hastily-written in a room, or a forest, or a disused building, or online). Not because I'm annoyed that no one made pre-Corona-style films for my entertainment during a worldwide crisis; because I believe it was a profoundly creatively-impaired period and it's a collective good to own it.

Everyone was reckoning with existential fears, and their limbic systems were loudly lit up in unfamiliar, uninteresting ways, and when I watch these films, I feel like I'm watching a guy lose the battle to acknowledge to himself that this just isn't going to be the year and he should tend to his own garden, instead of making a public demonstration of his ability to power through his stuck brain problems. As the audience, I feel dropped into the role of validator. Honestly, I feel like feature narrative film is simply too large, rigid, and wasteful a medium for the circumstances.

But, oh, the non-narrative short film. [Amyl and the Sniffers](https://www.amylandthesniffers.com) are a phenomenal punk band out of Melbourne, and [John Angus Stewart](http://www.johnangusstewart.com/info/) is a filmmaker from the same town who has made a bunch of their gorgeous videos. For me, this trio of shorts is the best audiovisual summation of the pandemic: what it's like being a little too intense and in an isolated feedback loop with your own energies, longing for connection:

1. [Guided by Angels](https://www.youtube.com/watch?v=Z--D1flPLnk)
2. [Hertz](https://www.youtube.com/watch?v=zb5Ja6V4OeY)
3. [Security](https://www.youtube.com/watch?v=Z--D1flPLnk)
