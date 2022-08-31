---
title: 'Getting To Grips With The Core Media IO Camera Extension Part 3 of 3: building a creative camera with realtime vImage Pixel Buffer effects using the Continuity Camera Webcam'
type: posts
tags: ['coremedia', 'swift', 'video', 'systemextensions']
categories: ['Development']
description: 'Bringing it all together by building a creative camera with realtime effects processing using vImage Pixel Buffers, which can use the Continuity Camera Webcam'
images: ['/images/cmio/NewWave.jpg']
date: 2022-08-31T08:00:00+01:00
---
# Getting To Grips With The Core Media IO Camera Extension, a 3 part series.

## Part 3 of 3: Bringing it all together by building a creative camera with realtime effects processing using vImage Pixel Buffers, which can use the Continuity Camera Webcam.

{{< datecodeanchorslug date="August 31st, 2022" >}}

#### [View project on Github](https://github.com/Halle/ArtFilm){#thecode}

Welcome to the third in a series of three posts about the **[Core Media IO Camera Extension](https://developer.apple.com/documentation/coremediaio/creating_a_camera_extension_with_core_media_i_o)**. 

The [first post, "**The Basics**"](https://theoffcuts.org/posts/core-media-io-camera-extensions-part-one/), is where you can learn what the series is about, its prerequisites, and to understand the project goals, while setting up the project for success.

The second was about extending that project by **[creating a useful software CMIO Camera Extension with communication between a configuration app and an extension, *and* learning how to do painless extension debugging](https://theoffcuts.org/posts/core-media-io-camera-extensions-part-two/)**,

This one is about bringing it all together by **building a creative camera with realtime effects processing using [vImage Pixel Buffers](https://developer.apple.com/documentation/accelerate/using_vimage_pixel_buffers_to_generate_video_effects), which can use the [Continuity Camera Webcam](https://developer.apple.com/videos/play/wwdc2022-10018)**.

The previous two posts supported macOS versions starting with 12.3, but this one is about new APIs in beta, so you will need to have a **Ventura beta** installed with a matching **Xcode beta**. I wrote them mostly with **Ventura beta 5**, so it is probably a good idea to have beta 5 or later. 

You'll also need a compatible beta OS on your phone in order to try the **Continuity Camera** features, but **Continuity Camera** isn't a requirement for working through the last part of this project – there will be fallback code to non-Continuity cameras, just like your extension should probably have, so no worries if you have an older phone, or a newer phone that you don't want to (or aren't allowed to) make a beta victim.

Let's jump in. To continue from where we left off, you can either [get my version](https://github.com/Halle/TechnicalDifficulties) of our previous results, keeping in mind that you will need to change all references to my organization ID (`com.politepix`) and my team ID to match your own, or ideally you have your own project that you made in the [previous post](https://theoffcuts.org/posts/core-media-io-camera-extensions-part-two/) and it is still working, so we can continue with it here.

We've dealt with all of the gnarly process topics like how to set up these projects correctly, how to install and uninstall extensions successfully, how very basic interprocess communication works for extensions, how to debug live extensions, and how to strenuously avoid debugging live extensions. With all of that packed away successfully, we can zero in on the fun topics today – live video and realtime effects.

## Loose ends

I first should tie up something I left open in my previous post. Right at the end, regarding a modification I made to the end-to-end testing app so that the technical difficulties image would display unmirrored, I wrote:

"I suspect my vertical-axis flipping in the end-to-end app isn’t correct for pixel buffers that originate from AVCaptureSession video, so I foresee a future improvement in which, instead of flipping the image, it complains when a pixel buffer was created with the wrong format or properties that would result in being in the wrong coordinate space once it gets to the camera system."

**Past me** was correct to be suspicious of this fix, but not for the right reasons. The reason the placard was mirrored was not because I wrote a bug, but because it should be mirrored. A system camera defaults to showing the user of the camera their own image **mirrored**, because that is how we're used to seeing ourselves. It shows them to the person on the other end of the video call **unmirrored**, because that is how others are used to seeing us. Which means that the mirrored image wasn't truly mirrored – when I made a test call and looked at the other end of the call, the image was shown correctly without my fix code. 

This means that the code added to the end-to-end testing app *is* correct and should remain in place, because it shows us what the enduser should see, but the fix added to the technical difficulties image in the extension code to unmirror it is not correct and should be removed, so that when we see the technical difficulties image in the end-to-end testing app, we see it mirrored, correctly.

## Picking up where we left off

Never mind the **[Technical Difficulties](https://www.github.com/Halle/TechnicalDifficulties)**, here's the **[Art Film](https://www.github.com/Halle/ArtFilm)**. Today we're going to do a bunch of different things: 

- Shift from a **software** camera which shows imagery from inside the extension, to a **creative** camera which alters a live camera feed, 
- Use that extremely fine new **Continuity Camera Webcam** as our video source,
- Do some direct interaction from the configuration app to the camera extension using **custom properties**, and with all of that working,
- Create realtime video effects for our creative camera using **vImage Pixel Buffers**, the new hotness for pixels + math from the **Accelerate** framework. 

We have a lot to get through and the changes will be extensive. Where possible, I am going to try to work with entire classes, but there will be a few line-by-line changes as we implement them. I am going to leave our work just in the four major files we've been using so far so that this doesn't become a tutorial about adding files to the correct target and which file to switch to, but please feel absolutely free to put each class or struct in its own file afterwards when you want to experiment further; I definitely would separate this into more files in my own ongoing project.

Throughout this process, you can always refer to the [completed sample app for this blog post at Github](http://github.com/Halle/ArtFilm).

## Some cleanup

The first thing we need to do is get ready to use a live camera and to use all the beta code in our three targets, so let's review and make sure that all three targets (extension, app, and end-to-end testing app) have a Camera usage sandbox capability, that all three targets have a `Privacy – Camera Usage Description` key in their `Info` settings with a valid description such as `"Camera Extension"`, and all three targets are set to deploy to macOS 13.0 (at least).

Next, please rename the file `NotificationName.swift` to `Shared.swift` because we are going to add some more shared Extension/App code there. 

Lastly, remove the jpeg files `Dirty.jpg` and `Clean.jpg`.

## Live camera streaming

We'll start by adding live camera streaming in our `ExtensionProvider` and making sure it works using the end-to-end testing app. We will do this using `AVCaptureSession`. Add the following imports to `Shared.swift`:

```
import AppKit
import AVFoundation
import CoreMediaIO
```

And now add this class to manage device frame capture to `Shared.swift`:

```
// MARK: - CaptureSessionManager

class CaptureSessionManager: NSObject {
    // MARK: Lifecycle

    init(capturingOffcutsCam: Bool) {
        super.init()
        captureOffcutsCam = capturingOffcutsCam
        configured = configureCaptureSession()
    }

    // MARK: Internal

    enum Camera: String {
        case continuityCamera
        case offcutsCam = "OffcutsCam"
    }

    var configured: Bool = false
    var captureOffcutsCam = false
    var captureSession: AVCaptureSession = .init()

    var videoOutput: AVCaptureVideoDataOutput?

    let dataOutputQueue = DispatchQueue(label: "video_queue",
                                        qos: .userInteractive,
                                        attributes: [],
                                        autoreleaseFrequency: .workItem)

    func configureCaptureSession() -> Bool {
        var result = false
        switch AVCaptureDevice.authorizationStatus(for: .video) {
        case .authorized:
            break
        case .notDetermined:
            AVCaptureDevice.requestAccess(
                for: .video,
                completionHandler: { granted in
                    if !granted {
                        logger.error("1. App requires camera access, returning")
                        return
                    } else {
                        result = self.configureCaptureSession()
                    }
                }
            )
            return result
        default:

            logger.error("2. App requires camera access, returning")
            return false
        }

        captureSession.beginConfiguration()

        captureSession.sessionPreset = sessionPreset

        guard let camera = getCameraIfAvailable(camera: captureOffcutsCam ? .offcutsCam : .continuityCamera) else {
            logger.error("Can't create default camera, this could be because the extension isn't installed, returning")
            return false
        }

        do {
            let fallbackPreset = AVCaptureSession.Preset.high
            let input = try AVCaptureDeviceInput(device: camera)
            
            let supportStandardPreset = input.device.supportsSessionPreset(sessionPreset)
            if !supportStandardPreset {
                let supportFallbackPreset = input.device.supportsSessionPreset(fallbackPreset)
                if supportFallbackPreset {
                    captureSession.sessionPreset = fallbackPreset
                } else {
                    logger.error("No HD formats used by this code supported, returning.")
                    return false
                }
            }
            captureSession.addInput(input)
        } catch {
            logger.error("Can't create AVCaptureDeviceInput, returning")
            return false
        }

        videoOutput = AVCaptureVideoDataOutput()
        if let videoOutput = videoOutput {
            if captureSession.canAddOutput(videoOutput) {
                captureSession.addOutput(videoOutput)
                captureSession.commitConfiguration()
                return true
            } else {
                logger.error("Can't add video output, returning")
                return false
            }
        }
        return false
    }

    // MARK: Private

    private let sessionPreset = AVCaptureSession.Preset.hd1280x720

    private func getCameraIfAvailable(camera: Camera) -> AVCaptureDevice? {
        let discoverySession = AVCaptureDevice.DiscoverySession(deviceTypes:
            [.externalUnknown, .builtInMicrophone, .builtInMicrophone, .builtInWideAngleCamera, .deskViewCamera],
            mediaType: .video, position: .unspecified)
        for device in discoverySession.devices {
            switch camera {
            case .continuityCamera:
                if device.isContinuityCamera, device.deviceType != .deskViewCamera {
                    return device
                }
            case .offcutsCam:
                if device.localizedName == camera.rawValue {
                    return device
                }
            }
        }
        return AVCaptureDevice.userPreferredCamera
    }
}
```

This is in `Shared.swift` because we will be using slightly different implementations of it in the extension and in the container app. Make sure all three targets can build and make any needed changes (overlooked imports, deployment versions, etc). 

Let's talk briefly about what is going on in this class and what it will do for us in `ExtensionProvider` once we implement it there.

1. In its `init()`, it finds out if it is supposed to capture the normal camera feed (e.g. the **Continuity Camera Webcam**, what we're using it for right now) or if we are going to use it to capture the feed from the installed extension (we'll do that later on).
2. It then does a very standard `AVCaptureSession` configuration, based on Apple's published best practices.
3. It tries to obtain the camera we specified (e.g. **Continuity**) by checking for a camera which is continuity but isn't the desk view, and returns it, or falls back to the `.userPreferredCamera` if that doesn't work out.
4. It sets up and adds the input and output of the session from this camera and if all is good, it commits the configuration and returns `true`, or logs errors and returns `false`. When we have set up the captureSession's final `sampleBufferDelegate` in whichever target is implementing this code, that target will receive the buffers from this configured `AVCaptureSession` and can do what it likes with them.

Now we can implement it in the extension code.

In `ExtensionProvider.swift`, add the following imports and constants:

```
import Accelerate
import AVFoundation

let outputWidth = 1280
let outputHeight = 720
```

Next, go to the `ExtensionStreamSource` function `init(localizedName: String, streamID: UUID, streamFormat: CMIOExtensionStreamFormat, device: CMIOExtensionDevice)` and add the following lines right under `super.init()`:

```
        captureSessionManager = CaptureSessionManager(capturingOffcutsCam: false)
        guard let captureSessionManager = captureSessionManager else {
            logger.error("Not able to get capture session, returning.")
            return
        }

        guard captureSessionManager.configured == true, let captureSessionManagerOutput = captureSessionManager.videoOutput else {
            logger.error("Not able to configure session and change captureSessionManagerOutput delegate, returning")
            return
        }
        captureSessionManagerOutput.setSampleBufferDelegate(self, queue: captureSessionManager.dataOutputQueue)
```

Add this variable to `ExtensionStreamSource `:

`private var captureSessionManager: CaptureSessionManager?`

Add the class conformance `AVCaptureVideoDataOutputSampleBufferDelegate` to `ExtensionStreamSource` in its class declaration right after `CMIOExtensionStreamSource` so we can a callback when there are captured buffers in the extension.

change the functions `func startStream()` and `func stopStream()` to this, so instead of starting the old streaming code they start and stop the `captureSession`:

```
    func startStream() throws {
        guard let captureSessionManager = captureSessionManager, captureSessionManager.captureSession.isRunning == false else {
            logger.error("Can't start capture session running, returning")
            return
        }
        captureSessionManager.captureSession.startRunning()
    }

    func stopStream() throws {
        guard let captureSessionManager = captureSessionManager, captureSessionManager.configured, captureSessionManager.captureSession.isRunning else {
            logger.error("Can't stop AVCaptureSession where it is expected, returning")
            return
        }
        if captureSessionManager.captureSession.isRunning {
            captureSessionManager.captureSession.stopRunning()
        }
    }
```

and add this function (this is the delegate callback we set up in the previous lines and conformance to receive captured `sampleBuffers` from the camera) to `ExtensionStreamSource` so we have captured buffers streaming in:

```

func captureOutput(_: AVCaptureOutput,
                       didOutput sampleBuffer: CMSampleBuffer,
                       from _: AVCaptureConnection)
    {
        guard let deviceSource = device.source as? ExtensionDeviceSource else {
            logger.error("Couldn't obtain device source")
            return
        }

        guard let pixelBuffer = sampleBuffer.imageBuffer else {
            return
        }

        CVPixelBufferLockBaseAddress(
            pixelBuffer,
            CVPixelBufferLockFlags.readOnly)

        var err: OSStatus = 0
        var sbuf: CMSampleBuffer!
        var timingInfo = CMSampleTimingInfo()
        timingInfo.presentationTimeStamp = CMClockGetTime(CMClockGetHostTimeClock())

        var formatDescription: CMFormatDescription?
        CMVideoFormatDescriptionCreateForImageBuffer(allocator: kCFAllocatorDefault, imageBuffer: pixelBuffer, formatDescriptionOut: &formatDescription)
        err = CMSampleBufferCreateReadyWithImageBuffer(allocator: kCFAllocatorDefault, imageBuffer: pixelBuffer, formatDescription: formatDescription!, sampleTiming: &timingInfo, sampleBufferOut: &sbuf)

        if err == 0 {
            if deviceSource._isExtension { // If I'm the extension, send to output stream
                stream.send(sbuf, discontinuity: [], hostTimeInNanoseconds: UInt64(timingInfo.presentationTimeStamp.seconds * Double(NSEC_PER_SEC)))
            } else {
                deviceSource.extensionDeviceSourceDelegate?.bufferReceived(sbuf) // If I'm the end to end testing app, send to delegate method.
            }
        } else {
            logger.error("Error in stream: \(err)")
        }

        CVPixelBufferUnlockBaseAddress(
            pixelBuffer,
            CVPixelBufferLockFlags.readOnly)
    }

```

remove the `private` designations from the variables `private var _isExtension: Bool = true` and `private var _videoDescription: CMFormatDescription!` and `private var _streamSource: ExtensionStreamSource!`

As you can see, we have moved the function where we handle streaming buffers from `ExtensionDeviceSource` to `ExtensionStreamSource`, as a consequence of the fact that we are starting and stopping the capture session in the stream (where it makes sense to do that) and that is also the class that gets the callback for the captured buffers. This feels natural to me, since these are all characteristics of the video stream.

Delete the contents of `func startStreaming()` in `ExtensionDeviceSource` but leave the function in place for now.

It should build. But, if we build and run the end-to-end testing app, we will get the "no video" image. This is because this more-complex extension requires some changes in the way the end-to-end app works so that it continues to emulate the system extension machinery. 

We'll make a couple of quick changes here. I am going to show you a more powerful way to communicate between the container app and extension later on, so let's repurpose our simple Darwin `CFNotification` system so that it is just for the end-to-end testing app. Add the following class to `Shared.swift`:

```

// MARK: - NotificationManager

class NotificationManager {
    class func postNotification(named notificationName: String) {
        let completeNotificationName = Identifiers.appGroup.rawValue + "." + notificationName
        logger
            .debug(
                "Posting notification \(completeNotificationName) from container app"
            )

        CFNotificationCenterPostNotification(
            CFNotificationCenterGetDarwinNotifyCenter(),
            CFNotificationName(completeNotificationName as NSString),
            nil,
            nil,
            true
        )
    }

    class func postNotification(named notificationName: NotificationName) {
        logger
            .debug(
                "Posting notification \(notificationName.rawValue) from container app"
            )

        CFNotificationCenterPostNotification(
            CFNotificationCenterGetDarwinNotifyCenter(),
            CFNotificationName(notificationName.rawValue as NSString),
            nil,
            nil,
            true
        )
    }
}
```
and replace the contents of the enum `enum NotificationName` so it looks like this:

```
enum NotificationName: String, CaseIterable {
    case startStream = "Z39BRGKSRW.com.politepix.OffcutsCam.startStream"
    case stopStream = "Z39BRGKSRW.com.politepix.OffcutsCam.stopStream"
}
```

And add this enum:

```
enum Identifiers: String {
    case appGroup = "Z39BRGKSRW.com.politepix.OffcutsCam"
    case orgIDAndProduct = "com.politepix.OffcutsCam"
}
```

Since we've added the identifiers, let's fix our two logger lines to reference them:

```
let logger = Logger(subsystem: Identifiers.orgIDAndProduct.rawValue.lowercased(),
                    category: "Application")
```   
and 

```
let logger = Logger(subsystem: Identifiers.orgIDAndProduct.rawValue.lowercased(),
                    category: "Extension")
```                                     

In `ExtensionProvider.swift`, remove the following notifications functions:

```

    private func notificationReceived(notificationName: String) {
        guard let name = NotificationName(rawValue: notificationName) else {
            return
        }

        switch name {
        case .changeImage:
            self.deviceSource.imageIsClean.toggle()
            logger.debug("The camera extension has received a notification")
            logger.debug("The notification is: \(name.rawValue)")
            self.deviceSource.stopStreaming()
            self.deviceSource.startStreaming()
        }
    }

    private func startNotificationListeners() {
        for notificationName in NotificationName.allCases {
            let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())

            CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), observer, { _, observer, name, _, _ in
                if let observer = observer, let name = name {
                    let extensionProviderSourceSelf = Unmanaged<ExtensionProviderSource>.fromOpaque(observer).takeUnretainedValue()
                    extensionProviderSourceSelf.notificationReceived(notificationName: name.rawValue as String)
                }
            },
            notificationName.rawValue as CFString, nil, .deliverImmediately)
        }
    }

    private func stopNotificationListeners() {
        if notificationListenerStarted {
            CFNotificationCenterRemoveEveryObserver(notificationCenter,
                                                    Unmanaged.passRetained(self)
                                                        .toOpaque())
            notificationListenerStarted = false
        }
    }
```

And then add this extension to the end of the file so we can support operating the `ExtensionProvider` with notifications when it isn't part of the system machinery, with better modularization as this file becomes more complex:

```

extension ExtensionProviderSource {
    private func notificationReceived(notificationName: String) {
        if let name = NotificationName(rawValue: notificationName) {
            switch name {
            case .startStream:
                do {
                    try deviceSource._streamSource.startStream()
                } catch {
                    logger.debug("Couldn't start the stream")
                }
            case .stopStream:
                do {
                    try deviceSource._streamSource.stopStream()
                } catch {
                    logger.debug("Couldn't stop the stream")
                }
            }
        }
    }

    private func startNotificationListeners() {
        var allNotifications = [String]()
        for notificationName in NotificationName.allCases {
            allNotifications.append(notificationName.rawValue)
        }

        for notificationName in allNotifications {
            let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())

            CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), observer, { _, observer, name, _, _ in
                if let observer = observer, let name = name {
                    let extensionProviderSourceSelf = Unmanaged<ExtensionProviderSource>.fromOpaque(observer).takeUnretainedValue()
                    extensionProviderSourceSelf.notificationReceived(notificationName: name.rawValue as String)
                }
            },
            notificationName as CFString, nil, .deliverImmediately)
        }
    }

    private func stopNotificationListeners() {
        if notificationListenerStarted {
            CFNotificationCenterRemoveEveryObserver(notificationCenter,
                                                    Unmanaged.passRetained(self)
                                                        .toOpaque())
            notificationListenerStarted = false
        }
    }
}

```
We have added a way to forward the end-to-end app's notifications into the `ExtensionProvider`, which lets us start the stream more like the system, now that we have improved the internal design of the `ExtensionProvider`.

This should now build and run the end-to-end testing app, but it won't change the behavior in the app until we use these new hooks in the end-to-end testing app's `ContentView.swift`.

Change the `EndToEndStreamProvider` `override init()` in the end-to-end testing app's `ContentView.swift` so it invokes the `ExtensionProvider'` `startStreaming()` call via notification:

```
    override init() {
        providerSource = ExtensionProviderSource(clientQueue: nil)
        super.init()
        providerSource
            .deviceSource = ExtensionDeviceSource(localizedName: "OffcutsCam")
        providerSource.deviceSource.extensionDeviceSourceDelegate = self

        NotificationManager
            .postNotification(
                named: NotificationName.startStream
            )
    }
```
Now, when you run the end-to-end testing app, you should see the Continuity Camera feed, or it should fall back to your preferred camera if there is no Continuity Camera. If you see nothing at all but you don't get an error, it is very important to make sure that your Continuity Camera, or your other camera if you aren't testing a Continuity Camera now, is definitely working in FaceTime. In the current Ventura betas, Continuity Camera can enter a weird state where the code finds it, and it reports a feed, but the feed is empty, and this prevents it from falling back to a working other camera or returning a "no camera" error, but no sample buffers are provided. I have seen this behavior several times, but never in a user session where the Continuity Camera Webcam was working normally in FaceTime. If it isn't working in FaceTime, restart your machine and your phone and try again.

We've gotten a live feed from the Continuity Camera and modified our end-to-end testing app so it can control our redesigned extension code, in order to maintain our painless debugging approach. Now that it works, you could also install the extension and reboot, if you felt like it. FaceTime should be able to use your camera afterwards, and it should behave like the Continuity Camera.

If everything is working so far, congrats! Get up and take a walk.

## Command and control

Back already? Cool, let's add some more direct control from the app to the extension so we can prepare for our special effects. In the last post, part 2, I explained how to use the very simple interprocess communication method of Darwin `CFNotification`, but now we will want more precise control, and in any case, we've retired `CFNotification` to usage by the end-to-end testing app exclusively. We will create custom properties for the camera extension that the container app can interface with directly.

First, it could be a good idea to watch [the part of the WWDC22 CMIO Camera Extension video](https://developer.apple.com/videos/play/wwdc2022-10022/?time=1637) about this, because it's vital context and it will also clarify why we're about to get some ugly wrappers.

## Ugly wrappers

CMIO Camera Extension custom properties are a C API. If you've done lower-level audio or video code, you know that sometimes we need wrappers to deal nicely with highly-performant or highly-lightweight C and C++ system APIs; if this is your first time, welcome to Gnarlyville! The main goal is to give yourself some tools so that you debug in a single place through a single interface for these translations, and then, with any luck, never think about it again.

First stop, still in Swift-land, add the following to `Shared.swift`, which will eventually control our special effects and give us a needed helpful extra `String` function:

```
// MARK: - MoodName

enum MoodName: String, CaseIterable {
    case bypass = "Bypass"
    case newWave = "New Wave"
    case berlin = "Berlin"
    case oldFilm = "OldFilm"
    case sunset = "Sunset"
    case badEnergy = "BadEnergy"
    case beyondTheBeyond = "BeyondTheBeyond"
    case drama = "Drama"
}

// MARK: - PropertyName

enum PropertyName: String, CaseIterable {
    case mood
}

extension String {
    func convertedToCMIOObjectPropertySelectorName() -> CMIOObjectPropertySelector {
        let noName: CMIOObjectPropertySelector = 0
        if count == MemoryLayout<CMIOObjectPropertySelector>.size {
            return data(using: .utf8, allowLossyConversion: false)?.withUnsafeBytes { propertySelector in
                propertySelector.load(as: CMIOObjectPropertySelector.self).byteSwapped
            } ?? noName
        } else {
            return noName
        }
    }
}

```
Open the `ContentView.swift` of the container app target (not the end-to-end app, which cannot directly access custom properties because it uses `ExtensionProvider` as part of its included code, rather than interacting with it as a system device whose service exposes properties). Add the following imports:

```
import AVFoundation
import CoreMediaIO
```

and then add the following class:


```
// MARK: - CustomPropertyManager

class CustomPropertyManager: NSObject {
    // MARK: Lifecycle

    override init() {
        super.init()
        effect = MoodName(rawValue: getPropertyValue(withSelectorName: mood) ?? MoodName.bypass.rawValue) ?? MoodName.bypass
    }

    // MARK: Internal

    let mood = PropertyName.mood.rawValue.convertedToCMIOObjectPropertySelectorName()
    var effect: MoodName = .bypass

    lazy var deviceObjectID: CMIOObjectID? = {
        let device = getExtensionDevice(name: "OffcutsCam")
        if let device = device, let deviceObjectId = getCMIODeviceID(fromUUIDString: device.uniqueID) {
            return deviceObjectId
        }
        return nil

    }()

    func getExtensionDevice(name: String) -> AVCaptureDevice? {
        let discoverySession = AVCaptureDevice.DiscoverySession(deviceTypes: [.externalUnknown],
                                                                mediaType: .video,
                                                                position: .unspecified)
        return discoverySession.devices.first { $0.localizedName == name }
    }

    func propertyExists(inDeviceAtID deviceID: CMIODeviceID, withSelectorName selectorName: CMIOObjectPropertySelector) -> CMIOObjectPropertyAddress? {
        var address = CMIOObjectPropertyAddress(mSelector: CMIOObjectPropertySelector(selectorName), mScope: CMIOObjectPropertyScope(kCMIOObjectPropertyScopeGlobal), mElement: CMIOObjectPropertyElement(kCMIOObjectPropertyElementMain))
        let exists = CMIOObjectHasProperty(deviceID, &address)
        return exists ? address : nil
    }

    func getCMIODeviceID(fromUUIDString uuidString: String) -> CMIOObjectID? {
        var propertyDataSize: UInt32 = 0
        var dataUsed: UInt32 = 0
        var cmioObjectPropertyAddress = CMIOObjectPropertyAddress(mSelector: CMIOObjectPropertySelector(kCMIOHardwarePropertyDevices), mScope: CMIOObjectPropertyScope(kCMIOObjectPropertyScopeGlobal), mElement: CMIOObjectPropertyElement(kCMIOObjectPropertyElementMain))
        CMIOObjectGetPropertyDataSize(CMIOObjectPropertySelector(kCMIOObjectSystemObject), &cmioObjectPropertyAddress, 0, nil, &propertyDataSize)
        let count = Int(propertyDataSize) / MemoryLayout<CMIOObjectID>.size
        var cmioDevices = [CMIOObjectID](repeating: 0, count: count)
        CMIOObjectGetPropertyData(CMIOObjectPropertySelector(kCMIOObjectSystemObject), &cmioObjectPropertyAddress, 0, nil, propertyDataSize, &dataUsed, &cmioDevices)
        for deviceObjectID in cmioDevices {
            cmioObjectPropertyAddress.mSelector = CMIOObjectPropertySelector(kCMIODevicePropertyDeviceUID)
            CMIOObjectGetPropertyDataSize(deviceObjectID, &cmioObjectPropertyAddress, 0, nil, &propertyDataSize)
            var deviceName: NSString = ""
            CMIOObjectGetPropertyData(deviceObjectID, &cmioObjectPropertyAddress, 0, nil, propertyDataSize, &dataUsed, &deviceName)
            if String(deviceName) == uuidString {
                return deviceObjectID
            }
        }
        return nil
    }

    func getPropertyValue(withSelectorName selectorName: CMIOObjectPropertySelector) -> String? {
        var propertyAddress = CMIOObjectPropertyAddress(mSelector: CMIOObjectPropertySelector(selectorName), mScope: CMIOObjectPropertyScope(kCMIOObjectPropertyScopeGlobal), mElement: CMIOObjectPropertyElement(kCMIOObjectPropertyElementMain))

        guard let deviceID = deviceObjectID else {
            logger.error("Couldn't get object ID, returning")
            return nil
        }

        if CMIOObjectHasProperty(deviceID, &propertyAddress) {
            var propertyDataSize: UInt32 = 0
            CMIOObjectGetPropertyDataSize(deviceID, &propertyAddress, 0, nil, &propertyDataSize)
            var name: NSString = ""
            var dataUsed: UInt32 = 0
            CMIOObjectGetPropertyData(deviceID, &propertyAddress, 0, nil, propertyDataSize, &dataUsed, &name)
            return name as String
        }
        return nil
    }

    func setPropertyValue(withSelectorName selectorName: CMIOObjectPropertySelector, to value: String) -> Bool {
        guard let deviceID = deviceObjectID, var propertyAddress = propertyExists(inDeviceAtID: deviceID, withSelectorName: selectorName) else {
            logger.debug("Property doesn't exist")
            return false
        }
        var settable: DarwinBoolean = false
        CMIOObjectIsPropertySettable(deviceID, &propertyAddress, &settable)
        if settable == false {
            logger.debug("Property can't be set")
            return false
        }
        var dataSize: UInt32 = 0
        CMIOObjectGetPropertyDataSize(deviceID, &propertyAddress, 0, nil, &dataSize)
        var changedValue: NSString = value as NSString
        let result = CMIOObjectSetPropertyData(deviceID, &propertyAddress, 0, nil, dataSize, &changedValue)
        if result != 0 {
            logger.debug("Not successful setting property data")
            return false
        }
        return true
    }
}
```
Let's run through this very quickly. After we create a `CustomerPropertyManager`, we can make get and set calls to the desired extension property selector, first obtaining the system extension object ID via a Swift string that is the name of the camera ("OffcutsCam") and then using that ID to query whether our selector exists on the object (i.e. find out if it has a published property matching our property "mood"), and if so, set or get it. Our selector is created from a four-letter string ("mood") and turned into a `CMIOObjectPropertySelector`, which is a `FourChar`. We use our extension on `String` from `Shared.swift` for that part. That's basically it.

When I hit issues with my property implementation, I used [this project for a sink Camera extension](https://github.com/ldenoue/cameraextension/blob/main/samplecamera/ViewController.swift) as a reference, so thank you [Laurent Denoue](https://github.com/ldenoue/) for providing a clear working example of property access so I could improve mine, especially given how time-consuming it can be to debug this area of an extension if it's unclear whether the issue is in the extension or the container app.

Let's clean up the view a little bit and make some changes. Replace all of `ContentView` with the following:   

```
struct ContentView {
    // MARK: Lifecycle
init(systemExtensionRequestManager: SystemExtensionRequestManager, propertyManager: CustomPropertyManager) {
        self.propertyManager = propertyManager
        self.systemExtensionRequestManager = systemExtensionRequestManager
        effect = moods.firstIndex(of: propertyManager.effect) ?? 0
    }

    // MARK: Internal

    var propertyManager: CustomPropertyManager
    @ObservedObject var systemExtensionRequestManager: SystemExtensionRequestManager

    // MARK: Private

    private var moods = MoodName.allCases
    @State private var effect: Int
}

// MARK: View

extension ContentView: View {
    var body: some View {
        VStack {
            Button("Install", action: {
                systemExtensionRequestManager.install()
            })
            Button("Uninstall", action: {
                systemExtensionRequestManager.uninstall()
            })

            Picker(selection: $effect, label: Text("Effect")) {
                ForEach(Array(moods.enumerated()), id: \.offset) { index, element in
                    Text(element.rawValue).tag(index)
                }
            }
            .pickerStyle(.segmented)

            .onChange(of: effect) { tag in
                let result = propertyManager.setPropertyValue(withSelectorName: propertyManager.mood, to: moods[tag].rawValue)
                logger.debug("Setting new property value (\"\(propertyManager.getPropertyValue(withSelectorName: propertyManager.mood) ?? "Unknown new string")\") was \(result ? "successful" : "unsuccessful")")
            }
            .disabled(propertyManager.deviceObjectID == nil)
            Text(systemExtensionRequestManager.logText)
        }
        .frame(alignment: .top)
        Spacer()
    }
}
```

And fix everything that initializes this view so it looks like this:

```
ContentView(systemExtensionRequestManager: SystemExtensionRequestManager(logText: ""), propertyManager: CustomPropertyManager())
```

Now, in `ExtensionProvider.swift`, in `ExtensionDeviceSource`, add this variable:

`var mood = MoodName.bypass`

and replace all the content of these property management functions of `ExtensionDeviceSource`:

- `var availableProperties: Set<CMIOExtensionProperty>`, 
- `func deviceProperties(forProperties properties: Set<CMIOExtensionProperty>) throws
        -> CMIOExtensionDeviceProperties`, and 
- `func setDeviceProperties(_ deviceProperties: CMIOExtensionDeviceProperties) throws` 

with the following block that includes returned data and behavior for our custom property:
        
```

    var availableProperties: Set<CMIOExtensionProperty> {
        [.deviceTransportType, .deviceModel, customEffectExtensionProperty]
    }

    func deviceProperties(forProperties properties: Set<CMIOExtensionProperty>) throws
        -> CMIOExtensionDeviceProperties
    {
        let deviceProperties = CMIOExtensionDeviceProperties(dictionary: [:])
        if properties.contains(.deviceTransportType) {
            deviceProperties.transportType = kIOAudioDeviceTransportTypeVirtual
        }
        if properties.contains(.deviceModel) {
            deviceProperties.model = "OffcutsCam Model"
        }

        // If I get there and there is a key for my effect, that means that we've run before.
        // We are backing the custom property with the extension's UserDefaults.
        let userDefaultsPropertyKey = PropertyName.mood.rawValue
        if userDefaults?.object(forKey: userDefaultsPropertyKey) != nil, let propertyMood = userDefaults?.string(forKey: userDefaultsPropertyKey) { // Not first run
            deviceProperties.setPropertyState(CMIOExtensionPropertyState(value: propertyMood as NSString),
                                              forProperty: customEffectExtensionProperty)

            if let moodName = MoodName(rawValue: propertyMood) {
                mood = moodName
            }
        } else { // We have never run before, so set property and the backing UserDefaults to default setting
            deviceProperties.setPropertyState(CMIOExtensionPropertyState(value: MoodName.bypass.rawValue as NSString),
                                              forProperty: customEffectExtensionProperty)
            userDefaults?.set(MoodName.bypass.rawValue, forKey: userDefaultsPropertyKey)
            logger.debug("Did initial set of effects value to \(MoodName.bypass.rawValue)")
            mood = MoodName.bypass
        }

        return deviceProperties
    }

    func setDeviceProperties(_ deviceProperties: CMIOExtensionDeviceProperties) throws {
        let userDefaultsPropertyKey = PropertyName.mood.rawValue
        if let customEffectValueFromPropertiesDictionary = dictionaryValueForEffectProperty(in: deviceProperties) {
            logger.debug("New setting in device properties for custom effect property: \(customEffectValueFromPropertiesDictionary)")
            userDefaults?.set(customEffectValueFromPropertiesDictionary, forKey: userDefaultsPropertyKey)
            if let moodName = MoodName(rawValue: customEffectValueFromPropertiesDictionary) {
                mood = moodName
            }
        }
    }     
        private let customEffectExtensionProperty: CMIOExtensionProperty = .init(rawValue: "4cc_" + PropertyName.mood.rawValue + "_glob_0000") // Custom 'effect' property

    private let userDefaults = UserDefaults(suiteName: Identifiers.appGroup.rawValue)

    private func dictionaryValueForEffectProperty(in deviceProperties: CMIOExtensionDeviceProperties) -> String? {
        guard let customEffectValueFromPropertiesDictionary = deviceProperties.propertiesDictionary[customEffectExtensionProperty]?.value as? String else {
            logger.debug("Was not able to get the value of the custom effect property from the properties dictionary of the device, returning.")
            return nil
        }
        return customEffectValueFromPropertiesDictionary
    }   
```

We have created a custom property called "mood". The entire property name is the string `"4cc_mood_glob_0000"` which is a standard prefix, the name of our property selector, the scope (global), and a standard suffix. In practice it means that there is a published selector for this extension which a client can connect to using the `FourChar` "mood", which should sound familiar after the code we added to the container app. This "mood" is eventually going to provide the extension with some information about which effect we want.

Bringing it all together, when we select an option in the container app picker, it obtains a reference to the device ID and this custom selector, and changes the value of the property, and the extension will react to this and also back the property change with an entry in its `UserDefaults` for persistence.  

In the container app UI, to debug this, every time the custom property is set in the UI, it also does a get so it can report the changed status to you.

I am very sorry to tell you that this is one part of the development process which requires reboots and reinstalls to troubleshoot, if properties aren't working quite right within your extension code. As mentioned, exposing the extension properties is a function of the system extension machinery, so there isn't any real debug feedback on this outside of that context. But I have already tested out this code, so if you follow these instructions, it shouldn't need debugging. At this point, you should be able to build, run, install the new extension from the container app, quit the app, build and run the app again, and see feedback in the debugger when you make selections. Between app sessions, your choices will persist. 

One very important other thing, so we don't bring all of our fast debugging to a halt with this improvement: let's add some code to the end-to-end testing app `ContentView` that will at least allow us to test the *consequences* of property changes from the end-to-end testing app, even if it can't access the actual properties. Add these variables to the end-to-end testing app's `ContentView`:

```
@State var effect: Int
var moods = MoodName.allCases
```
and add this familiar-looking picker in the view:

```
  Picker(selection: $effect, label: Text("Effect")) {
                ForEach(Array(moods.enumerated()), id: \.offset) { index, element in
                    Text(element.rawValue).tag(index)
                }
            }
            .pickerStyle(.segmented)

            .onChange(of: effect) { tag in
                logger.debug("Chosen effect: \(moods[tag].rawValue)")
                NotificationManager.postNotification(named: moods[tag].rawValue)
            }
```

And change how the ContentView is initialized, anywhere you need to:

```
ContentView(endToEndStreamProvider: EndToEndStreamProvider(), effect: 0)

```
Change our `ExtensionProvider` extension's function `private func startNotificationListeners()` to this:

```
    private func startNotificationListeners() {
        var allNotifications = [String]()
        for notificationName in NotificationName.allCases {
            allNotifications.append(notificationName.rawValue)
        }

        for notificationName in MoodName.allCases {
            allNotifications.append(Identifiers.appGroup.rawValue + "." + notificationName.rawValue)
        }

        for notificationName in allNotifications {
            let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())

            CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), observer, { _, observer, name, _, _ in
                if let observer = observer, let name = name {
                    let extensionProviderSourceSelf = Unmanaged<ExtensionProviderSource>.fromOpaque(observer).takeUnretainedValue()
                    extensionProviderSourceSelf.notificationReceived(notificationName: name.rawValue as String)
                }
            },
            notificationName as CFString, nil, .deliverImmediately)
        }
    }
```

and change its `private func notificationReceived(notificationName: String)` to this:

```
    private func notificationReceived(notificationName: String) {
        if let name = NotificationName(rawValue: notificationName) {
            switch name {
            case .startStream:
                do {
                    try deviceSource._streamSource.startStream()
                } catch {
                    logger.debug("Couldn't start the stream")
                }
            case .stopStream:
                do {
                    try deviceSource._streamSource.stopStream()
                } catch {
                    logger.debug("Couldn't stop the stream")
                }
            }
        } else {
            if let mood = MoodName(rawValue: notificationName.replacingOccurrences(of: Identifiers.appGroup.rawValue + ".", with: "")) {
                deviceSource.mood = mood
            }
        }
    }
```

Which will forward the same `mood` info into `ExtensionProvider` when there are no properties to interact with because we're running it directly in the end-to-end testing app.

Maybe make a nice cup of tea? See you in a bit.

## Effects!

Oh, hello! We're finally ready for the effects. Please download the following 7 images and add them to your `Extension` target and your `OffcutsCamEndToEnd` target. Please make **extra** sure that they are added to both.

- [1.jpg](/images/cmio/moods/1.jpg)
- [2.jpg](/images/cmio/moods/2.jpg)
- [3.jpg](/images/cmio/moods/3.jpg)
- [4.jpg](/images/cmio/moods/4.jpg)
- [5.jpg](/images/cmio/moods/5.jpg)
- [6.jpg](/images/cmio/moods/6.jpg)
- [7.jpg](/images/cmio/moods/7.jpg)

[(Image licensing)](https://raw.githubusercontent.com/Halle/ArtFilm/main/image_licenses.txt)

Confirm that they have been successfully added by building the container app and listing the contents of its directory `OffcutsCam.app/Contents/Library/SystemExtensions/com.politepix.OffcutsCam.Extension.systemextension/Contents/Resources/`, (replacing my bundle ID with yours) which is where they should be.
 
Add the following import and variable to the top of `ExtensionProvider`:

```
import AppKit

let pixelBufferSize = vImage.Size(width: outputWidth, height: outputHeight)
```
 
and set `let kFrameRate: Int = 1` to `let kFrameRate: Int = 24`.
 
Add this class to `ExtensionProvider`:

```

// MARK: - Effects

class Effects: NSObject {
    // MARK: Lifecycle

    // Effect processing with vImage Pixel Buffers

    override init() {
        super.init()
        if let image = NSImage(named: "1.jpg") { // Get histograms for all the chosen images in init
            sourceImageHistogramNewWave = getHistogram(for: image)
        }
        if let image = NSImage(named: "2.jpg") {
            sourceImageHistogramBerlin = getHistogram(for: image)
        }
        if let image = NSImage(named: "3.jpg") {
            sourceImageHistogramOldFilm = getHistogram(for: image)
        }
        if let image = NSImage(named: "4.jpg") {
            sourceImageHistogramSunset = getHistogram(for: image)
        }
        if let image = NSImage(named: "5.jpg") {
            sourceImageHistogramBadEnergy = getHistogram(for: image)
        }
        if let image = NSImage(named: "6.jpg") {
            sourceImageHistogramBeyondTheBeyond = getHistogram(for: image)
        }
        if let image = NSImage(named: "7.jpg") {
            sourceImageHistogramDrama = getHistogram(for: image)
        }

        let randomNumberGenerator = BNNSCreateRandomGenerator(
            BNNSRandomGeneratorMethodAES_CTR,
            nil)!

        for _ in 0 ..< maximumNoiseArrays { // Get random noise for all the noise buffers in init
            let noiseBuffer = vImage.PixelBuffer(
                size: pixelBufferSize,
                pixelFormat: vImage.InterleavedFx3.self)

            let shape = BNNS.Shape.tensor3DFirstMajor(
                noiseBuffer.width,
                noiseBuffer.height,
                noiseBuffer.channelCount)

            noiseBuffer.withUnsafeMutableBufferPointer { noisePtr in

                if var descriptor = BNNSNDArrayDescriptor(
                    data: noisePtr,
                    shape: shape) {
                    let mean: Float = 0.0125
                    let stdDev: Float = 0.025

                    BNNSRandomFillNormalFloat(randomNumberGenerator, &descriptor, mean, stdDev)
                }
            }
            noiseBufferArray.append(noiseBuffer)
        }
    }

    // MARK: Internal

    let cvImageFormat = vImageCVImageFormat.make(
        format: .format422YpCbCr8,
        matrix: kvImage_ARGBToYpCbCrMatrix_ITU_R_601_4.pointee,
        chromaSiting: .center,
        colorSpace: CGColorSpaceCreateDeviceRGB(),
        alphaIsOpaqueHint: true)!

    var cgImageFormat = vImage_CGImageFormat(
        bitsPerComponent: 32,
        bitsPerPixel: 32 * 3,
        colorSpace: CGColorSpaceCreateDeviceRGB(),
        bitmapInfo: CGBitmapInfo(
            rawValue: CGBitmapInfo.byteOrder32Little.rawValue |
                CGBitmapInfo.floatComponents.rawValue |
                CGImageAlphaInfo.none.rawValue),
        renderingIntent: .defaultIntent)!

    let destinationBuffer = vImage.PixelBuffer(
        size: pixelBufferSize,
        pixelFormat: vImage.InterleavedFx3.self)

    func populateDestinationBuffer(pixelBuffer: CVPixelBuffer) { // Convert into destinationBuffer content
        let sourceBuffer = vImage.PixelBuffer(
            referencing: pixelBuffer,
            converter: converter,
            destinationPixelFormat: vImage.DynamicPixelFormat.self)

        do {
            try converter.convert(
                from: sourceBuffer,
                to: destinationBuffer)
        } catch {
            fatalError("Any-to-any conversion failure.")
        }
    }

    func artFilm(forMood mood: MoodName) { // Apply mood to frame
        tastefulNoise(destinationBuffer: destinationBuffer)
        specifySavedHistogram(forMood: mood)
        mildTemporalBlur()
    }

    // MARK: Private

    private lazy var converter: vImageConverter = {
        guard let converter = try? vImageConverter.make(
            sourceFormat: cvImageFormat,
            destinationFormat: cgImageFormat)
        else {
            fatalError("Unable to create converter")
        }

        return converter
    }()

    private lazy var temporalBuffer = vImage.PixelBuffer( // temporal blur storage
        size: pixelBufferSize,
        pixelFormat: vImage.InterleavedFx3.self)

    private lazy var histogramBuffer = vImage.PixelBuffer( // Temp histogram storage
        size: pixelBufferSize,
        pixelFormat: vImage.PlanarFx3.self)

    private var noiseBufferArray: [vImage.PixelBuffer<vImage.InterleavedFx3>] = .init()

    private var sourceImageHistogramNewWave: vImage.PixelBuffer.HistogramFFF? // All set up before applying effect
    private var sourceImageHistogramBerlin: vImage.PixelBuffer.HistogramFFF?
    private var sourceImageHistogramOldFilm: vImage.PixelBuffer.HistogramFFF?
    private var sourceImageHistogramSunset: vImage.PixelBuffer.HistogramFFF?
    private var sourceImageHistogramBadEnergy: vImage.PixelBuffer.HistogramFFF?
    private var sourceImageHistogramBeyondTheBeyond: vImage.PixelBuffer.HistogramFFF?
    private var sourceImageHistogramDrama: vImage.PixelBuffer.HistogramFFF?

    private let maximumNoiseArrays = kFrameRate /
        2 // How many noise arrays we'll use for faking continuous random noise
    private var noiseArrayCount = 0
    private var noiseArrayCountAscending = true
    private let histogramBinCount = 32

    private func mildTemporalBlur() {
        let interpolationConstant: Float = 0.4

        destinationBuffer.linearInterpolate(
            bufferB: temporalBuffer,
            interpolationConstant: interpolationConstant,
            destination: temporalBuffer)

        temporalBuffer.copy(to: destinationBuffer)
    }

    private func tastefulNoise(destinationBuffer: vImage.PixelBuffer<vImage.InterleavedFx3>) {
        guard noiseBufferArray.count == maximumNoiseArrays else {
            return
        }

        destinationBuffer.withUnsafeMutableBufferPointer { mutableDestintationPtr in
            vDSP.add(destinationBuffer, noiseBufferArray[noiseArrayCount],
                     result: &mutableDestintationPtr)
        }

        if noiseArrayCount == maximumNoiseArrays - 1 {
            noiseArrayCountAscending = false
        } else if noiseArrayCount == 0 {
            if noiseArrayCountAscending == false {
                // the maximumNoiseArrays * 2 pass, we shuffle so the eyes don't start to notice patterns in the "noise dance"
                noiseBufferArray = noiseBufferArray.shuffled()
            }
            noiseArrayCountAscending = true
        }

        if noiseArrayCountAscending {
            noiseArrayCount += 1
        } else {
            noiseArrayCount -= 1
        }
    }

    private func specifySavedHistogram(forMood mood: MoodName) {
        var sourceHistogramToSpecify = sourceImageHistogramNewWave

        switch mood { // Choose from among our pre-populated histograms
        case .newWave:
            sourceHistogramToSpecify = sourceImageHistogramNewWave
        case .berlin:
            sourceHistogramToSpecify = sourceImageHistogramBerlin
        case .oldFilm:
            sourceHistogramToSpecify = sourceImageHistogramOldFilm
        case .sunset:
            sourceHistogramToSpecify = sourceImageHistogramSunset
        case .badEnergy:
            sourceHistogramToSpecify = sourceImageHistogramBadEnergy
        case .beyondTheBeyond:
            sourceHistogramToSpecify = sourceImageHistogramBeyondTheBeyond
        case .drama:
            sourceHistogramToSpecify = sourceImageHistogramDrama
        case .bypass:
            return
        }

        if let sourceImageHistogram = sourceHistogramToSpecify {
            destinationBuffer.deinterleave(
                destination: histogramBuffer)

            histogramBuffer.specifyHistogram(
                sourceImageHistogram,
                destination: histogramBuffer)

            histogramBuffer.interleave(
                destination: destinationBuffer)
        }
    }

    private func getHistogram(for image: NSImage) -> vImage.PixelBuffer.HistogramFFF? { // Extract histogram from image
        let sourceImageHistogramBuffer = vImage.PixelBuffer(
            size: pixelBufferSize,
            pixelFormat: vImage.PlanarFx3.self)

        var proposedRect = NSRect(
            origin: CGPoint(x: 0.0, y: 0.0),
            size: CGSize(width: image.size.width, height: image.size.height))

        guard let cgImage = image.cgImage(forProposedRect: &proposedRect, context: nil, hints: nil) else {
            logger.error("Couldn't get cgImage from \(image), returning.")
            return nil
        }

        let bytesPerPixel = cgImage.bitsPerPixel / cgImage.bitsPerComponent
        let destBytesPerRow = outputWidth * bytesPerPixel

        guard let colorSpace = cgImage.colorSpace, let context = CGContext(
            data: nil,
            width: outputWidth,
            height: outputHeight,
            bitsPerComponent: cgImage.bitsPerComponent,
            bytesPerRow: destBytesPerRow,
            space: colorSpace,
            bitmapInfo: cgImage.alphaInfo.rawValue) else {
            logger.error("Problem setting up cgImage resize, returning.")
            return nil
        }

        context.interpolationQuality = .none
        context.draw(cgImage, in: CGRect(x: 0, y: 0, width: outputWidth, height: outputHeight))

        guard let resizedCGImage = context.makeImage() else {
            logger.error("Couldn't resize cgImage for histogram, returning.")
            return nil
        }

        let pixelFormat = vImage.InterleavedFx3.self

        let sourceImageBuffer: vImage.PixelBuffer<vImage.InterleavedFx3>?

        do {
            sourceImageBuffer = try vImage.PixelBuffer(
                cgImage: resizedCGImage,
                cgImageFormat: &cgImageFormat,
                pixelFormat: pixelFormat)

            if let sourceImageBuffer = sourceImageBuffer {
                sourceImageBuffer.deinterleave(destination: sourceImageHistogramBuffer)
                return sourceImageHistogramBuffer.histogram(binCount: histogramBinCount)
            } else {
                logger.error("Source image buffer was nil, returning.")
                return nil
            }
        } catch {
            logger.error("Error creating source image buffer: \(error)")
            return nil
        }
    }
}

```

Well, this is a very exciting class. Let me talk a little bit about the goals, and then I will describe what is going on in here.

So, what I wanted to do was to create a very filmic, very color-pushed visual effect with noise, maybe a little like an [Anton Corbijn image in color](https://youtu.be/u1xrNaTO1bI?t=20), but much more unreal. I was thinking about how you could extract a histogram from a still image and apply it to another still in `Accelerate vImage`, and decided that I would like to combine this with the new `vImage Pixel Buffers` functions by taking several still images I thought were interesting, extracting their histograms at initialization, keeping the histograms in variables, and then applying them (specifying them) to our streaming video frames in realtime.

I also need to apply some noise, both for stylistic reasons but also because it is the only way that these forced histograms can dither enough to draw gradients in shot backgrounds – otherwise they would just draw solid color blocks every time there is a "missing" color in the histogram that would otherwise be needed to bridge two colors in a gradient. To get my noise, I am referencing Apple's fantastic sample app [Using vImage Pixel Buffers to Generate Video Effects](https://developer.apple.com/documentation/accelerate/using_vimage_pixel_buffers_to_generate_video_effects), with a caveat, which is: I don't think we should ever be using a function like this to generate fresh noise at every buffer callback. I'm sure that Apple doesn't intend for developers to do that; they are just showing how performant it is, and how to implement it. But, if someone were to do that in a camera, it would use a lot of fuel doing something that human perception can't memorize in a detailed way over time, so we should be fooling those humans with something that just looks like fresh noise.

This means, much like with my histograms, I am filling up a set number of buffers with generated `vImage` noise at initialization, and applying them to the realtime video stream buffers in a way that makes them look random: I go forward through the array, I go backwards through the array, then I shuffle the array, then I do it again. This prevents noise pattern recognition, where the sequence of noise appears to do a preset "dance" at a regular interval. But it means that whether you talk for a minute or an hour, the same amount of energy was spent on generating the noise buffers. If you think it's too frugal and you can see it when you pixel-peep, try a larger number. I would like to find a way to optimize my histogram specification more as well, because without the live noise generation, that part is now the biggest resource user.

Last, because it is a crowd-pleaser, if my Twitter feed is anything to go by, I included a very small amount of temporal blur just to make it frea-- uh, to create a filmic motion blur? Sure, that's it. Unlike the noise function, the temporal blur is very resource-frugal and doesn't need optimization or tricks, so it can just be run similarly to how Apple does it in their example (but much like my noise constants, I am using a lot less of the temporal blur because it is **A Lot**, and it can get disturbing with a low color palette and low framerate). 

The last thing affecting the look of the video feed is that we turn the framerate down to 24.

This is not a prettification filter; I promised in the last post that we were going to get satisfyingly weird in this post and I keep my promises. This is, like, for Contrapoints to use while addressing Congress via video feed on a giant screen, or for Blixa Bargeld to use while making the school carpool arrangements.

It's also a nice one for a tutorial because you can easily try it out with different jpegs of your own, change the noise and temporal constants, etc.

There are seven images, representing seven "moods" or looks.

The basic process of the filters are similar. Buffers are specified at initialization with the `vImage Pixel Buffer` buffer type they are going to need in the callback. We have a `converter` we will be able to use to take our camera feed `CVPixelBuffer` and turn it into our `vImage Pixel Buffer` `destinationBuffer`, which we can then do our `vImage` operations on. Each of the three effect passes starts and ends with a prepared `destinationBuffer`, and when we're done, we will convert `destinationBuffer` back into a preconfigured `CVPixelBuffer` and `send()` it out to our stream.

Ready? Let's plug it into `ExtensionStreamSource`. Give `ExtensionStreamSource` some new variables:

```
let effects = Effects()
var destinationCVPixelBuffer: CVPixelBuffer?
var deviceSource: ExtensionDeviceSource?
```

and add these lines to its `init(localizedName: String, streamID: UUID, streamFormat: CMIOExtensionStreamFormat, device: CMIOExtensionDevice)` right under `super.init()` to set up a `destinationCVPixelBuffer` in advance, to receive the post-effects frames before dispatch to the stream output:

```

        self.deviceSource = device.source as? ExtensionDeviceSource

        guard let deviceSource = deviceSource else {
            logger.error("No device source, returning")
            return
        }

        let pixelBufferAttributes: NSDictionary = [
            kCVPixelBufferWidthKey: outputWidth,
            kCVPixelBufferHeightKey: outputHeight,
            kCVPixelBufferPixelFormatTypeKey: deviceSource._videoDescription.mediaSubType,
            kCVPixelBufferIOSurfacePropertiesKey: [:],
        ]

        let result = CVPixelBufferCreate(kCFAllocatorDefault,
                                         outputWidth,
                                         outputHeight,
                                         kCVPixelFormatType_422YpCbCr8,
                                         pixelBufferAttributes as CFDictionary,
                                         &destinationCVPixelBuffer)

        if result != 0 {
            logger.error("Couldn't create destination buffer, returning")
            return
        }
```

Replace our good old sampleBufferDelegate callback:

`func captureOutput(_: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from _: AVCaptureConnection)` 

with this version that uses our effect buffers and effects:
                       
```
func captureOutput(_: AVCaptureOutput,
                       // Callback for sampleBuffers of captured video, which we apply our effects to in realtime
                       didOutput sampleBuffer: CMSampleBuffer,
                       from _: AVCaptureConnection) {
        guard let pixelBuffer = sampleBuffer.imageBuffer, let deviceSource = deviceSource,
              let destinationCVPixelBuffer = destinationCVPixelBuffer else {
            logger.debug("Nothing to do in sampleBuffer callback, returning.")
            return
        }

        CVPixelBufferLockBaseAddress(
            pixelBuffer,
            CVPixelBufferLockFlags.readOnly)

        effects.populateDestinationBuffer(pixelBuffer: pixelBuffer)
        if deviceSource.mood != .bypass {
            effects.artFilm(forMood: deviceSource.mood)
        }

        CVPixelBufferUnlockBaseAddress(
            pixelBuffer,
            CVPixelBufferLockFlags.readOnly)

        var err: OSStatus = 0
        var sbuf: CMSampleBuffer!
        var timingInfo = CMSampleTimingInfo()
        timingInfo.presentationTimeStamp = CMClockGetTime(CMClockGetHostTimeClock())

        CVPixelBufferLockBaseAddress(destinationCVPixelBuffer,
                                     CVPixelBufferLockFlags(rawValue: 0))

        do {
            try effects.destinationBuffer.copy(
                to: destinationCVPixelBuffer,
                cvImageFormat: effects.cvImageFormat,
                cgImageFormat: effects.cgImageFormat)
        } catch {
            logger.error("Copying to the destinationBuffer failed.")
        }

        CVPixelBufferUnlockBaseAddress(destinationCVPixelBuffer, CVPixelBufferLockFlags(rawValue: 0))

        var formatDescription: CMFormatDescription?
        CMVideoFormatDescriptionCreateForImageBuffer(
            allocator: kCFAllocatorDefault,
            imageBuffer: destinationCVPixelBuffer,
            formatDescriptionOut: &formatDescription)
        err = CMSampleBufferCreateReadyWithImageBuffer(
            allocator: kCFAllocatorDefault,
            imageBuffer: destinationCVPixelBuffer,
            formatDescription: formatDescription!,
            sampleTiming: &timingInfo,
            sampleBufferOut: &sbuf) // CVPixelBuffer into CMSampleBuffer for streaming out

        if err == 0 {
            if deviceSource._isExtension { // If I'm the extension, send to output stream
                stream.send(
                    sbuf,
                    discontinuity: [],
                    hostTimeInNanoseconds: UInt64(timingInfo.presentationTimeStamp.seconds * Double(NSEC_PER_SEC)))
            } else {
                deviceSource.extensionDeviceSourceDelegate?
                    .bufferReceived(sbuf) // If I'm the end to end testing app, send to delegate method.
            }
        } else {
            logger.error("Error in stream: \(err)")
        }
    }
```  

You can now completely remove the following functions and variables from `ExtensionProvider.swift`:
                     
`var imageIsClean = true`    

`func pixelBufferFromImage(_ image: CGImage) -> CVPixelBuffer?`

`private var _whiteStripeStartRow: UInt32 = 0`

`private var _whiteStripeIsAscending: Bool = false`

`func startStreaming()`

`func stopStreaming()`

` private var _streamingCounter: UInt32 = 0`

`private var _timer: DispatchSourceTimer?`

`private let _timerQueue = DispatchQueue(label: "timerQueue", qos: .userInteractive, attributes: [], autoreleaseFrequency: .workItem, target: .global(qos: .userInteractive))`
                                            
That's it for the extension. You should be able to run it in the end-to-end app now and try out its filters. The only thing we're missing is a video monitor in the container app. As I mentioned in the first post, we are only writing UIs in **SwiftUI** in these posts, and not using `NSViewRepresentable`, so let's reuse our `CaptureSessionManager` in combination with our IOSurface->CGImage approach from the end-to-end app to give ourselves a feed of **OffcutsCam** when the extension is installed.

Add this class to the container app's `ContentView.swift`:

```

// MARK: - OutputImageManager

class OutputImageManager: NSObject, AVCaptureVideoDataOutputSampleBufferDelegate, ObservableObject {
    @Published var videoExtensionStreamOutputImage: CGImage?
    let noVideoImage: CGImage = NSImage(
        systemSymbolName: "video.slash",
        accessibilityDescription: "Image to indicate no video feed available"
    )!.cgImage(forProposedRect: nil, context: nil, hints: nil)! // OK to fail if this isn't available.

    func captureOutput(_: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from _: AVCaptureConnection) {
        autoreleasepool {
            guard let cvImageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else {
                logger.debug("Couldn't get image buffer, returning.")
                return
            }

            guard let ioSurface = CVPixelBufferGetIOSurface(cvImageBuffer) else {
                logger.debug("Pixel buffer had no IOSurface") // The camera uses IOSurface so we want image to break if there is none.
                return
            }

            let ciImage = CIImage(ioSurface: ioSurface.takeUnretainedValue())
                .oriented(.upMirrored) // Cameras show the user a mirrored image, the other end of the conversation an unmirrored image.

            let context = CIContext(options: nil)

            guard let cgImage = context
                .createCGImage(ciImage, from: ciImage.extent) else { return }

            DispatchQueue.main.async {
                self.videoExtensionStreamOutputImage = cgImage
            }
        }
    }
}
```

This will receive the output buffers from a properly-configured `AVCaptureSession` and convert them to a CGImage. Change the `init()` of `ContentView` to this:

```
    init(systemExtensionRequestManager: SystemExtensionRequestManager, propertyManager: CustomPropertyManager, outputImageManager: OutputImageManager) {
        self.propertyManager = propertyManager
        self.systemExtensionRequestManager = systemExtensionRequestManager
        self.outputImageManager = outputImageManager
        effect = moods.firstIndex(of: propertyManager.effect) ?? 0
        captureSessionManager = CaptureSessionManager(capturingOffcutsCam: true)

        if captureSessionManager.configured == true, captureSessionManager.captureSession.isRunning == false {
            captureSessionManager.captureSession.startRunning()
            captureSessionManager.videoOutput?.setSampleBufferDelegate(outputImageManager, queue: captureSessionManager.dataOutputQueue)
        } else {
            logger.error("Couldn't start capture session")
        }
    }
```

and add these variables to `ContentView`:

```
@ObservedObject var outputImageManager: OutputImageManager    
var captureSessionManager: CaptureSessionManager
```

in the `View` of `ContentView`, add this image:

```
            Image(
                self.outputImageManager
                    .videoExtensionStreamOutputImage ?? self.outputImageManager
                    .noVideoImage,
                scale: 1.0,
                label: Text("Video Feed")
            )
```

Wherever that `ContentView` is invoked, change the call to this:

```
ContentView(systemExtensionRequestManager: SystemExtensionRequestManager(logText: ""), propertyManager: CustomPropertyManager(), outputImageManager: OutputImageManager())
```

Let's add this to both our `VStacks` in the `ContentView` of the container app and the end-to-end testing app so our layout is nicer:

```
.frame(alignment: .top)
Spacer()
```        

And let's add this frame to `ContentView()` everywhere that a `ContentView` is created:

```
.frame(minWidth: 1280, maxWidth: 1360, minHeight: 900, maxHeight: 940)
```

And now you should be able to see your installed extension content in an image at the top of your container app. When installing the extension for the first time, it is necessary to quit and restart the container app before this will work.

I encountered one persistent bug in this phase of this post, which is that sometimes, the `/Applications` version of `OffcutsCam.app` would complain when asked to install the extension that it had an invalid codesign. In these cases, it was necessary to select `Xcode->Product->Show Build Folder in Finder` and move the copy of `OffcutsCam.app` in there to `/Applications` manually, at which point the error stopped.

I encountered a milder bug (but still something that would be very stressful if I were discovering this the hard way via a tutorial not working) which was that I had one installation where the effects didn't work in the container app<->extension interaction but they did work in the end-to-end app, which self-healed after a second install, restart, reinstall. Well, we know that we're using betas, and some mysteries are part of the beta experience. 

That's everything! This has been quite a journey. I hope you feel well-set-up to start experimenting with your own effects and creative camera experiences. If you've had any trouble, compare against my [version on Github](https://github.com/Halle/ArtFilm) and I'm sure you'll find the issue in no time. Have fun!

## Extro

✂ - - Non-code thoughts, please stop reading here for the code-only experience - - ✂

Extro will follow up in a few days after publication, as usual.