---
title: 'Prototyping a stationary bike "step" counter with Swift Charts'
type: posts
tags: ['swiftui', 'swift charts', 'coremotion', 'prototyping']
categories: ['Development']
description: 'Prototyping a stationary bike "step" counter with Swift Charts'
images: ['/images/gyro2.png']
date: 2022-07-11T12:00:00+01:00
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

{{< highlighter color="yellow" ranges="11-11, 35-36, 54-54, 77-77, 119-124, 159-159, 230-233">}}
// Created by Halle Winkler on July/11/22. Copyright © 2022. All rights reserved.
// Requires Xcode 14.x and iOS 16.x, betas included.

import Charts
import CoreMotion
import SwiftUI

// MARK: - ContentView

/// ContentView is a collection of motion sensor UIs and a method of calling back to the model.

struct ContentView {
    @ObservedObject var manager: MotionManager
}

extension ContentView: View {
    var body: some View {
        VStack {
            ForEach(manager.sensors, id: \.sensorName) { sensor in
                SensorChart(sensor: sensor) { applyFilter, lowPassFilterFactor, quantizeFactor in
                    manager.updateFilteringFor(
                        sensor: sensor,
                        applyFilter: applyFilter,
                        lowPassFilterFactor: lowPassFilterFactor,
                        quantizeFactor: quantizeFactor)
                }
            }
        }.padding([.leading, .trailing], 6)
    }
}

// MARK: - SensorChart

/// I like to compose SwiftUI interfaces out of many small modules. But, there is a tension when it's a
/// small UI overall, and the modules will each have overhead from propagating state, binding and callbacks.

struct SensorChart {
    @State private var chartIsVisible = true
    @State private var breakOutAxes = false
    @State private var applyingFilter = false
    @State private var lowPassFilterFactor: Double = 0.75
    @State private var quantizeFactor: Double = 50
    var sensor: Sensor
    let updateFiltering: (Bool, Double, Double) -> Void
    private func toggleFiltering() {
        applyingFilter.toggle()
        updateFiltering(applyingFilter, lowPassFilterFactor, quantizeFactor)
    }
}

extension SensorChart: View {
    var body: some View {
/// Per-sensor controls: apply filtering to the waveform, hide and show sensor, break out the axes into separate charts.

        HStack {
            Text("\(sensor.sensorName)")
                .font(.system(size: 12, weight: .semibold, design: .default))
                .foregroundColor(chartIsVisible ? .black : .gray)
            Spacer()
            Button(action: toggleFiltering) {
                Image(systemName: applyingFilter ? "waveform.circle.fill" :
                    "waveform.circle")
            }
            .opacity(chartIsVisible ? 1.0 : 0.0)
            Button(action: { chartIsVisible.toggle() }) {
                Image(systemName: chartIsVisible ? "eye.circle.fill" :
                    "eye.slash.circle")
            }
            Button(action: { breakOutAxes.toggle() }) {
                Image(systemName: breakOutAxes ? "1.circle.fill" :
                    "3.circle.fill")
            }
            .opacity(chartIsVisible ? 1.0 : 0.0)
        }

/// Sensor charts, either one chart with three axes, or three charts with one axis. I love how concise Swift Charts can be.

        if chartIsVisible {
            if breakOutAxes {
                ForEach(sensor.axes, id: \.axisName) { series in
                    // Iterate charts from series
                    Chart {
                        ForEach(
                            Array(series.measurements.enumerated()),
                            id: \.offset) { index, datum in
                                LineMark(
                                    x: .value("Count", index),
                                    y: .value("Measurement", datum))
                            }
                    }
                    Text(
                        "Axis: \(series.axisName)\(applyingFilter ? "\t\tPeaks in window: \(series.peaks)" : "")")
                }
                .chartXAxis {
                    AxisMarks(values: .automatic(desiredCount: 0))
                }
            } else {
                Chart {
                    ForEach(sensor.axes, id: \.axisName) { series in
                        // Iterate series in a chart
                        ForEach(
                            Array(series.measurements.enumerated()),
                            id: \.offset) { index, datum in
                                LineMark(
                                    x: .value("Count", index),
                                    y: .value("Measurement", datum))
                            }
                            .foregroundStyle(by: .value("MeasurementName",
                                                        series.axisName))
                    }
                }.chartXAxis {
                    AxisMarks(values: .automatic(desiredCount: 0))
                }.chartYAxis {
                    AxisMarks(values: .automatic(desiredCount: 2))
                }
            }

/// in the separate three-axis view, you can set the low-pass filter factor and the quantizing factor if the waveform
/// filtering is on, and then once you can see your stationary pedaling reflected in the waveform, you can see how
/// many times per time window you're pedaling. With such an inevitably-noisy sensor environment, I already know
/// the low-pass filter factor will have to be very high, so I'm starting it at 0.75.
/// In the case of my exercise bike, the quantizing factor  that delivers very accurate peak-counting results on
/// gyroscope axis z is 520, which tells you these readings are really small numbers.

            if applyingFilter {
                Slider(
                    value: $lowPassFilterFactor,
                    in: 0.75 ... 1.0,
                    onEditingChanged: { _ in
                        updateFiltering(
                            true,
                            lowPassFilterFactor,
                            quantizeFactor)
                    })
                Text("Lowpass: \(String(format: "%.2f", lowPassFilterFactor))")
                    .font(.system(size: 12))
                    .frame(width: 100, alignment: .trailing)
                Slider(
                    value: $quantizeFactor,
                    in: 1 ... 600,
                    onEditingChanged: { _ in
                        updateFiltering(
                            true,
                            lowPassFilterFactor,
                            quantizeFactor)
                    })
                Text("Quantize: \(Int(quantizeFactor))")
                    .font(.system(size: 12))
                    .frame(width: 100, alignment: .trailing)
            }
        }
        Divider()
    }
}

// MARK: - MotionManager

/// MotionManager is the sensor management module.

class MotionManager: ObservableObject {
    // MARK: Lifecycle

    init() {
        self.manager = CMMotionManager()
        for name in SensorNames
            .allCases {
// self.sensors and func collectReadings(...) use SensorNames to index,
            if name ==
                .attitude {
// so if you change how one creates/derives a sensor index, change them both.
                sensors.append(ThreeAxisReadings(
                    sensorName: SensorNames.attitude.rawValue,
                    // The one exception to sensor axis naming:
                    axes: [
                        Axis(axisName: "Pitch"),
                        Axis(axisName: "Roll"),
                        Axis(axisName: "Yaw"),
                    ]))
            } else {
                sensors.append(ThreeAxisReadings(sensorName: name.rawValue))
            }
        }
        self.manager.deviceMotionUpdateInterval = sensorUpdateInterval
        self.manager.accelerometerUpdateInterval = sensorUpdateInterval
        self.manager.gyroUpdateInterval = sensorUpdateInterval
        self.manager.magnetometerUpdateInterval = sensorUpdateInterval
        self.startDeviceUpdates(manager: manager)
    }

    // MARK: Public

    public func updateFilteringFor( // Manage the callbacks from the UI
        sensor: ThreeAxisReadings,
        applyFilter: Bool,
        lowPassFilterFactor: Double,
        quantizeFactor: Double) {
        guard let index = sensors.firstIndex(of: sensor) else { return }
        DispatchQueue.main.async {
            self.sensors[index].applyFilter = applyFilter
            self.sensors[index].lowPassFilterFactor = lowPassFilterFactor
            self.sensors[index].quantizeFactor = quantizeFactor
        }
    }

    // MARK: Internal

    struct ThreeAxisReadings: Equatable {
        var sensorName: String // Usually, these have the same naming:
        var axes: [Axis] = [Axis(axisName: "x"), Axis(axisName: "y"),
                            Axis(axisName: "z")]
        var applyFilter: Bool = false
        var lowPassFilterFactor = 0.75
        var quantizeFactor = 1.0

        func lowPassFilter(lastReading: Double?, newReading: Double) -> Double {
            guard let lastReading else { return newReading }
            return self
                .lowPassFilterFactor * lastReading +
                (1.0 - self.lowPassFilterFactor) * newReading
        }
    }

    struct Axis: Hashable {
        var axisName: String
        var measurements: [Double] = []
        var peaks = 0
        var updatesSinceLastPeakCount = 0

/// I love sets, like, a lot. Enough that when I first thought "but what's an *elegant* way to know when it's a
/// good time to count the peaks again?" I thought of a one-liner set intersection, very semantic, very accurate to the
/// underlying question of freshness of sensor data, and it made me happy, and I smiled.
/// Anyway, a counter does the same thing with a 0s execution time, here's one of those:

        mutating func shouldCountPeaks()
            -> Bool { // Peaks are only counted once a second
            updatesSinceLastPeakCount += 1
            if updatesSinceLastPeakCount == MotionManager.updatesPerSecond {
                updatesSinceLastPeakCount = 0
                return true
            }
            return false
        }
    }

    @Published var sensors: [ThreeAxisReadings] = []

    // MARK: Private

    private enum SensorNames: String, CaseIterable {
        case attitude = "Attitude"
        case rotationRate = "Rotation Rate"
        case gravity = "Gravity"
        case userAcceleration = "User Acceleration"
        case acceleration = "Acceleration"
        case gyroscope = "Gyroscope"
        case magnetometer = "Magnetometer"
    }

    private static let updatesPerSecond: Int = 30

    private let motionQueue = OperationQueue() // Don't read sensors on main

    private let secondsToShow = 5 // Time window to observe
    private let sensorUpdateInterval = 1.0 / Double(updatesPerSecond)
    private let manager: CMMotionManager

    private func startDeviceUpdates(manager _: CMMotionManager) {
        self.manager
            .startDeviceMotionUpdates(to: motionQueue) { motion, error in
                self.collectReadings(motion, error)
            }
        self.manager
            .startAccelerometerUpdates(to: motionQueue) { motion, error in
                self.collectReadings(motion, error)
            }
        self.manager.startGyroUpdates(to: motionQueue) { motion, error in
            self.collectReadings(motion, error)
        }
        self.manager
            .startMagnetometerUpdates(to: motionQueue) { motion, error in
                self.collectReadings(motion, error)
            }
    }

    private func collectReadings(_ motion: CMLogItem?, _ error: Error?) {
        DispatchQueue.main.async { // Add new readings on main
            switch motion {
            case let motion as CMDeviceMotion:
                self.appendReadings(
                    [motion.attitude.pitch, motion.attitude.roll,
                     motion.attitude.yaw],
                    to: &self.sensors[SensorNames.attitude.index()])
                self.appendReadings(
                    [motion.rotationRate.x, motion.rotationRate.y,
                     motion.rotationRate.z],
                    to: &self.sensors[SensorNames.rotationRate.index()])
                self.appendReadings(
                    [motion.gravity.x, motion.gravity.y, motion.gravity.z],
                    to: &self.sensors[SensorNames.gravity.index()])
                self.appendReadings(
                    [motion.userAcceleration.x, motion.userAcceleration.y,
                     motion.userAcceleration.z],
                    to: &self.sensors[SensorNames.userAcceleration.index()])
            case let motion as CMAccelerometerData:
                self.appendReadings(
                    [motion.acceleration.x, motion.acceleration.y,
                     motion.acceleration.z],
                    to: &self.sensors[SensorNames.acceleration.index()])
            case let motion as CMGyroData:
                self.appendReadings(
                    [motion.rotationRate.x, motion.rotationRate.y,
                     motion.rotationRate.z],
                    to: &self.sensors[SensorNames.gyroscope.index()])
            case let motion as CMMagnetometerData:
                self.appendReadings(
                    [motion.magneticField.x, motion.magneticField.y,
                     motion.magneticField.z],
                    to: &self.sensors[SensorNames.magnetometer.index()])
            default:
                print(error != nil ? "Error: \(String(describing: error))" :
                    "Unknown device")
            }
        }
    }

    private func appendReadings(
        _ newReadings: [Double],
        to threeAxisReadings: inout ThreeAxisReadings) {
        for index in 0 ..< threeAxisReadings.axes
            .count { // For each of the axes
            var axis = threeAxisReadings.axes[index]
            let newReading = newReadings[index]

            axis.measurements
                .append(threeAxisReadings
                    .applyFilter ? // Append new reading, as-is or filtered
                    threeAxisReadings.lowPassFilter(
                        lastReading: axis.measurements.last,
                        newReading: newReading) : newReading)

            if threeAxisReadings.applyFilter,
               axis
               .shouldCountPeaks() {
                // And occasionally count peaks if filtering
                axis.peaks = countPeaks(
                    in: axis.measurements,
                    quantizeFactor: threeAxisReadings.quantizeFactor)
            }

            if axis.measurements
                .count >=
                Int(1.0 / self
                    .sensorUpdateInterval * Double(self.secondsToShow)) {
                axis.measurements
                    .removeFirst() // trim old data to keep our moving window representing secondsToShow
            }
            threeAxisReadings.axes[index] = axis
        }
    }

    private func countPeaks(
        in readings: [Double],
        quantizeFactor: Double) -> Int { // Count local maxima
        let quantizedreadings = readings.map { Int($0 * quantizeFactor) }
        // Quantize into small Ints (instead of extremely small Doubles) to remove detail from little component waves

        var ascendingWave = true
        var numberOfPeaks = 0
        var lastReading = 0

        for reading in quantizedreadings {
            if ascendingWave == true,
               lastReading >
               reading { // If we were going up but it stopped being true,
                numberOfPeaks += 1 // we just passed a peak,
                ascendingWave = false // and we're going down.
            } else if lastReading <
                reading {
                // If we just started to or continue to go up, note we're ascending.
                ascendingWave = true
            }
            lastReading = reading
        }
        return numberOfPeaks
    }
}

extension CaseIterable where Self: Equatable {
    func index() -> Self.AllCases
        .Index {
        // Force-unwrap of index of enum case in CaseIterable always succeeds.
        return Self.allCases.firstIndex(of: self)!
    }
}

typealias Sensor = MotionManager.ThreeAxisReadings
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
