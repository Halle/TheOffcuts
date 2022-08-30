---
title: 'Getting To Grips With The Core Media IO Camera Extension Part 1 of 3: The Basics'
type: posts
tags: ['coremedia', 'swift', 'video', 'systemextensions']
categories: ['Development']
description: 'Creating an installable Core Media Camera Extension and its container app'
images: ['/images/cmio/TechnicalDifficulties.jpg']
date: 2022-08-11T12:00:00+01:00
---
# Getting To Grips With The Core Media IO Camera Extension, a 3 part series.

## Part 1 of 3: The Basics: creating an installable Core Media Camera Extension and its container app.
{{< datecodeanchorslug date="August 11th, 2022" >}}

#### [View project on Github](https://github.com/Halle/OffcutsCam){#thecode}

Welcome to the first in a series of three posts about the **[Core Media IO Camera Extension](https://developer.apple.com/documentation/coremediaio/creating_a_camera_extension_with_core_media_i_o)**. I was delighted by the [Create camera extensions with Core Media IO
](https://developer.apple.com/videos/play/wwdc2022-10022) WWDC22 video, which is about extensions which can present a system camera for use in any camera-supporting app such as FaceTime, including creative cameras that can take in a feed from an existing camera such as a [Continuity Camera Webcam](https://developer.apple.com/videos/play/wwdc2022/10018/) and add effects to it. Still, I wished that it had sample code for the two types of cameras that it discussed, software camera and creative camera with configuration app. 

I wrote myself some sample code, and now I would like to share it, and explain a little about how it works. 

There will be three posts in the series: 

The first post, this one, is about **getting the basics working from the start**,

the second is about [**creating a useful software CMIO Camera Extension with communication between a configuration app and an extension *and* painless extension debugging**](https://theoffcuts.org/posts/core-media-io-camera-extensions-part-two/),

and the last is about bringing it all together by **building a creative camera with realtime effects processing using [vImage Pixel Buffers](https://developer.apple.com/documentation/accelerate/using_vimage_pixel_buffers_to_generate_video_effects), which can use the Continuity Camera Webcam**.

Each will build on the previous post. SwiftUI will be the only interface framework used, and there will be no `NSViewRepresentable`, which is going to get spicy in the final entry when we're observing a camera feed in the configuration app.

### Prerequisites

Much of this will work in macOS 12.3 and later with Xcode 13, so my [sample apps for part 1](https://github.com/Halle/OffcutsCam) and [2](https://github.com/Halle/TechnicalDifficulties) will build, run and work with them. But, the end result will explore beta APIs, so this series as a whole has been written for the betas. To run all of the code in all three parts:

* Ventura beta 5 or later
* Xcode 14 beta 5 or later
* An iPhone XS or later running iOS 16 beta 5 or later if you want to test a Continuity Camera Webcam camera extension in later parts of this series
* An Apple Developer account so you have a team ID and can codesign, which is already working correctly on your system when you sign other apps. If you've had some codesign weirdness you've been ignoring, this isn't the project for working through it, trust me.
* It is certainly a good idea to have looked at the [WWDC video](https://developer.apple.com/videos/play/wwdc2022-10022) and the [docs](https://developer.apple.com/documentation/coremediaio/creating_a_camera_extension_with_core_media_i_o), but it isn't a requirement.

Let's jump in. Here are the instructions for creating a container app and installable CMIO Camera Extension which is a basic software camera (you can also [clone mine from Github or refer to it](https://github.com/Halle/OffcutsCam) as you go):


### Project configuration

1. Create a new project of type "macOS App". Title it "OffcutsCam", select your team and organization identifier. Your organization identifier will be different than mine.
___
![](/images/cmio/1a.png)
___
![](/images/cmio/1b.png)
___
2. Go to the app target, select **Signing & Capabilities**, and remove "Hardened Runtime". Under **App Sandbox**, check "Camera" and, for debugging purposes only, set "User Selected File" to "Read Only".
___
![](/images/cmio/2a.png)
___
![](/images/cmio/2b.png)
___
3. Click the "+" button at the top left to add a **capability**. Add "System Extension".
___
![](/images/cmio/3.png)
___
4. Now add a target to your project (`File->New->Target`). Scroll all the way down in the target types to choose "Camera Extension". Title it "Extension"
___
![](/images/cmio/4a.png)
___
![](/images/cmio/4b.png)
___
5. Go to the new **extension** target, select **Signing & Capabilities**, and remove "Hardened Runtime". Under **App Sandbox**, check "Camera" and, for debugging purposes only, set "User Selected File" to "Read Only".
___
![](/images/cmio/5a.png)
___
![](/images/cmio/5b.png)
___
6. See that the **extension** target has an **App Group**. Change the extension target app group to `$(TeamIdentifierPrefix)com.politepix.OffcutsCam` but use your organization identifier instead of mine (com.politepix). Copy this app group name.
___
![](/images/cmio/6.png)
___
7. Now go back to your **app** target, **Signing & Capabilities**, and add the capability "App Group". In the new configuration area this adds, click "+" and add an app group with the identical app group name as the extension `($(TeamIdentifierPrefix)com.politepix.OffcutsCam` (but with your organization identifier instead of mine).
___
![](/images/cmio/7.png)
___
8. Still in your app target, go to **Info** and add the key `Privacy - Camera Usage Description` with a description of `Camera Extension` and while you're here, just to avoid a distracting warning later, add the key `App Category` and set it to `Utilities`.
___
![](/images/cmio/8a.png)
___
![](/images/cmio/8b.png)
___
9. Switching to your **extension** target, go to **Info** and add the key `Privacy - Camera Usage Description` with a description of `Camera Extension`. There should be a key entitled `Privacy - System Extension Usage Description` (if there isn't, create it). Add the description `Camera Extension` to it.
___
![](/images/cmio/9a.png)
___
![](/images/cmio/9b.png)
___
10. In the **extension** target, under **Info**, there should be a key `CMIOExtension` which is a dictionary. It should contain a key `CMIOExtensionMachServiceName`. The value of this key should be `$(TeamIdentifierPrefix)$(PRODUCT_BUNDLE_IDENTIFIER)`. Change it to `$(TeamIdentifierPrefix)com.politepix.OffcutsCam `(but with your organization identifier instead of mine).
___
![](/images/cmio/10.png)
___
Now, theoretically, doing these steps in this order should result in a properly-configured Container App and Embedded System Extension, where the container app is allowed to install its embedded system extension into macOS. To verify this, you can check the following things in your entitlements files:

App entitlements should have a `System Extension` key set to `YES`. It should have an `App Groups` array with the first element the string `$(TeamIdentifierPrefix)com.politepix.OffcutsCam` but with your organization identifier instead of mine. It should have a key `Camera` set to `YES`. Here is a screenshot:
___
![](/images/cmio/AppEntitlements.png)
___
Extension entitlements should have an identical `App Groups` array. It should have a key `Camera` set to `YES`. Here is a screenshot:
___
![](/images/cmio/ExtensionEntitlements.png)
___
The app target's **General** pane should show the extension as embedded "(Embed Without Signing)" under **Frameworks, Libraries, and Embedded Content**.
___
![](/images/cmio/embed.png)
___
If any of these aren't right, review and see if you set things up correctly. You can compare to my [completed version](https://github.com/Halle/OffcutsCam).

If this looks good, you can build and run. You should see a "Hello, world!" app. You can quit it. Go to `Xcode->Product->Show Build Folder in Finder` and find your product `OffcutsCam.app`, and right-click and choose `Show Package Contents` the package and verify that you can see the extension inside of it like in this screenshot.
___
![](/images/cmio/Package.png)
___

OK, **stretch your legs for a moment** and we'll start configuring the extension and app source. 

Ready?

### Source configuration

1. Open `ExtensionProvider.swift` in the editor. This is where the input and output streams for the camera extension are managed. Apple is kind enough to provide a 100% known-working software camera in all fresh ExtensionProviders. I love that they do this.
2. Do a search and replace in this file for every occurrence of `SampleCapture` and change it to `OffcutsCam`. Change the occurrence of `OffcutsCam (Swift)` to just `OffcutsCam`. This is how we are letting the system and the user know which camera this is. 

### Inside the CMIOExtensionProvider

Let's talk briefly about what is going on inside of `ExtensionProvider.swift`. At the system level, this is the code that the OS extension machinery is going to engage with to provide a camera, starting with initializing its service in `Main.swift`:

```
let providerSource = ExtensionProviderSource(clientQueue: nil)
CMIOExtensionProvider.startService(provider: providerSource.provider)
```

The `CMIOExtensionProvider` itself has three essential parts: 
- The `CMIOExtensionProviderSource`, a protocol where the conforming class manages the capabilities and properties which affect the entire extension, and where client (video app) connections to the provider (camera extension) are managed, and which creates the object conforming to:
- `CMIOExtensionDeviceSource`, a protocol where the conforming class manages the properties which affect the camera device specifically (meaning, the software device or the camera video feed which provides the basis for a creative camera), which configures the object conforming to:
- the `CMIOExtensionStreamSource`, a protocol where the conforming class manages the stream mechanics of the camera, such as starting and stopping a stream, and manages the properties of the stream. 

This is also the order the objects which conform to these protocols are initialized in, in an installed extension when there is a client that wants to use the camera. 

When we replaced the default names with "OffcutsCam", we replaced the manufacturer name in `CMIOExtensionProviderSource` and initialized the `deviceSource` object with its device name, we set the device model name in its property of `CMIOExtensionDeviceSource`, and we set the name of the stream where our `CMIOExtensionDeviceSource` object creates its `CMIOExtensionStreamSource` object.

Apple's default `ExtensionProvider` code demonstrates the responsibilities of each of these three parts, and the white line business code (`func startStreaming()`) can be found in the object conforming to `CMIOExtensionDeviceSource` before its final buffer is passed to the object conforming to `CMIOExtensionStreamSource` so it is seen in the client app using the camera extension. The business code that is eventually invoked by the `startStream()` function of `CMIOExtensionStreamSource` can get very different in practice, which we'll explore more in part 3 of this series.

That's it for `ExtensionProvider.swift`, for this post (there will be much more `ExtensionProvider` in the following two posts). But for now, we want to get this default software camera extension fully working so we have a clean canvas to paint on.

### Installation/deinstallation

Next, we will use Brad Ford's onscreen sample code for extension install and uninstall from [Create camera extensions with Core Media IO
](https://developer.apple.com/videos/play/wwdc2022-10022), adding it to our container app. First, add `import SystemExtensions` at the top of the App's file `ContentView.swift`.

Then, add the following class to `ContentView.swift` (my only addition here was more verbose error logging):

```
class SystemExtensionRequestManager: NSObject, ObservableObject {
    @Published var logText: String = "Installation results here"

    init(logText: String) {
        super.init()
        self.logText = logText
    }

    func install() {
        guard let extensionIdentifier = _extensionBundle().bundleIdentifier else { return }
        let activationRequest = OSSystemExtensionRequest.activationRequest(forExtensionWithIdentifier: extensionIdentifier, queue: .main)
        activationRequest.delegate = self
        OSSystemExtensionManager.shared.submitRequest(activationRequest)
    }

    func uninstall() {
        guard let extensionIdentifier = _extensionBundle().bundleIdentifier else { return }
        let deactivationRequest = OSSystemExtensionRequest.deactivationRequest(forExtensionWithIdentifier: extensionIdentifier, queue: .main)
        deactivationRequest.delegate = self
        OSSystemExtensionManager.shared.submitRequest(deactivationRequest)
    }

    func _extensionBundle() -> Bundle {
        let extensionsDirectoryURL = URL(fileURLWithPath: "Contents/Library/SystemExtensions", relativeTo: Bundle.main.bundleURL)
        let extensionURLs: [URL]
        do {
            extensionURLs = try FileManager.default.contentsOfDirectory(at: extensionsDirectoryURL, includingPropertiesForKeys: nil, options: .skipsHiddenFiles)
        } catch {
            fatalError("failed to get the contents of \(extensionsDirectoryURL.absoluteString): \(error.localizedDescription)")
        }
        guard let extensionURL = extensionURLs.first else {
            fatalError("failed to find any system extensions")
        }
        guard let extensionBundle = Bundle(url: extensionURL) else {
            fatalError("failed to create a bundle with URL \(extensionURL.absoluteString)")
        }
        return extensionBundle
    }
}

extension SystemExtensionRequestManager: OSSystemExtensionRequestDelegate {
    public func request(_ request: OSSystemExtensionRequest, actionForReplacingExtension existing: OSSystemExtensionProperties, withExtension ext: OSSystemExtensionProperties) -> OSSystemExtensionRequest.ReplacementAction {
        logText = "Replacing extension version \(existing.bundleShortVersion) with \(ext.bundleShortVersion)"
        return .replace
    }

    public func requestNeedsUserApproval(_ request: OSSystemExtensionRequest) {
        logText = "Extension needs user approval"
    }

    public func request(_ request: OSSystemExtensionRequest, didFinishWithResult result: OSSystemExtensionRequest.Result) {
        switch result.rawValue {
        case 0:
            logText = "\(request) did finish with success"
        case 1:
            logText = "\(request) Extension did finish with result success but requires reboot"
        default:
            logText = "\(request) Extension did finish with result \(result)"
        }
    }

    public func request(_ request: OSSystemExtensionRequest, didFailWithError error: Error) {
        let errorCode = (error as NSError).code
        var errorString = ""
        switch errorCode {
        case 1:
            errorString = "unknown error"
        case 2:
            errorString = "missing entitlement"
        case 3:
        	errorString = "Container App for Extension has to be in /Applications to install Extension."
        case 4:
            errorString = "extension not found"
        case 5:
            errorString = "extension missing identifier"
        case 6:
            errorString = "duplicate extension identifer"
        case 7:
            errorString = "unknown extension category"
        case 8:
            errorString = "code signature invalid"
        case 9:
            errorString = "validation failed"
        case 10:
            errorString = "forbidden by system policy"
        case 11:
            errorString = "request canceled"
        case 12:
            errorString = "request superseded"
        case 13:
            errorString = "authorization required"
        default:
            errorString = "unknown code"
        }
        logText = "Extension did fail with error: \(errorString)"
    }


}
```
in your `ContentView`, add this variable:
```
@ObservedObject var systemExtensionRequestManager: SystemExtensionRequestManager
```

and this view content:

```
     VStack {
            Button("Install", action: {
                systemExtensionRequestManager.install()
            })
            Button("Uninstall", action: {
                systemExtensionRequestManager.uninstall()
            })
        }
        Text(systemExtensionRequestManager.logText)
    }
     
```                       
                            
Change the calls in the container app that load ContentView to load it with its new `SystemExtensionRequestManager` and let's give the frame a minimum size:
                            
```
ContentView(systemExtensionRequestManager: SystemExtensionRequestManager(logText: "Starting container app"))
.frame(minWidth: 300, minHeight: 180)
```                                 
 
Now build and run the app again. You should see a UI with an **`Install`** and an **`Uninstall`** button. 

Click **`Install`** and see that you get an informative error that the container app isn't in `/Applications`. System extensions have to be installed from a container app in `/Applications` to be acceptable to the system.

Let's make things easy on ourselves and set things up so the builds are moved to `/Applications` so we can install the codesigned extension like an enduser would, but without too much bother. 

Let's edit the scheme for OffcutsCam.app. In the left column under `Build`, which we can open with the disclosure triangle, add a Post-Action "Run Script Action" script reading `ditto "${CODESIGNING_FOLDER_PATH}" "/Applications/${FULL_PRODUCT_NAME}"`. This will copy our app into `/Applications` once all building and signing is complete. 
___
![](/images/cmio/scheme.png)
___

Close the scheme, and build and run. Check `/Applications` and verify that `OffcutsCam.app` is in there.

Next, edit the app scheme a second time, and this time choose `Run`, and change the Executable. Select "Other", and then navigate to your `/Applications` folder and select your build of `OffcutsCam.app` that is now there. We've told the debugger to attach to our copy in `/Applications` instead of the one in `DerivedData`.
___
![](/images/cmio/scheme2.png)
___
With these steps complete, if you build and run `OffcutsCam.app` (we never need to run the extension directly), you should be able to click **`Install`** in the app UI and do an installation of your extension. The app should log that the extension needs user approval and macOS should show an alert saying "System Extension Blocked". That's great! That's how it's supposed to work.

___
![](/images/cmio/approval.png)
___
![](/images/cmio/blocked.png)
___

Click "Open System Settings" on that alert (if you clicked "OK" instead that's fine, go ahead and open the System Settings.app section "Security & Privacy"). Scroll down until you see the notification `System software from application "OffcutsCam" was blocked from loading` and click `Allow`.

___
![](/images/cmio/notification.png)
___

Authenticate to install. Now the extension should be installed in your system. You can verify this by opening FaceTime. Under the **Video** menu you should see **OffcutsCam** as an available camera. If you select it, you should see a black screen with a white line moving up and down it. Congrats! Your first CMIO Camera Extension.

We have taken pains to get codesigned extension installation working from the start so that we don't need to debug this area of the project while debugging other complexities later on such as interprocess communication and realtime video processing.

And that brings us to the conclusion of part one of this series: we have created a **Container App** which can install and uninstall a working **Core Media IO Camera System Extension** that can be selected in FaceTime, and we have removed one big pain point already, which is testing this behavior with full codesigning while still being able to do a normal build and run.

My working version can be seen [here](https://github.com/Halle/OffcutsCam), so if you are getting weird results, you can compare them.

Once you have things working, if you want to play with the extension provider code (and if you like video code, you probably do), be aware that to see software changes in your extension, you currently have to uninstall the old extension using your container app, then reboot, then build and run your new container app iteration, then install the changed extension. This is also currently the case if you follow the [debugging advice](https://developer.apple.com/documentation/driverkit/debugging_and_testing_system_extensions) for loading a new extension iteration without reversioning it (i.e., turning off SIP and turning on developer mode will not help with this). If I understand correctly, there is no way currently to securely replace the extension process in a single user session in which there has already been an extension install and activation, so a reboot is necessary.

This reboot-a-rama is the pain point we are going to remove next, in [**Core Media IO Camera Extensions part 2 of 3: Creating a useful software CMIO Camera Extension with communication between a configuration app and an extension *and* painless extension debugging**](https://theoffcuts.org/posts/core-media-io-camera-extensions-part-two/).

## Extro

✂ - - Non-code thoughts, please stop reading here for the code-only experience - - ✂

Since we're combining system extensions, video code, and codesigning, i.e. discussing pain, I thought it might be a good opportunity for some stoic philosophy.

There were three big stoics whose work we know about; [Seneca](https://plato.stanford.edu/entries/seneca/), Epictetus and Marcus Aurelius. I'm not crazy about Seneca. [Marcus Aurelius](https://plato.stanford.edu/entries/marcus-aurelius/) had some very helpful things to say, but he was an extraordinarily powerful man who has become a sort of aspirational lifestyle stoic: philosopher most likely to be cited in a LinkedIn post. For me, [Epictetus](https://plato.stanford.edu/entries/epictetus/) is the most interesting. An ancient Greek philosopher, a onetime slave, someone who had to avoid anti-philosopher purges now and again – you're reading a blog post about camera extensions so it's pretty likely that you and I are in the same lucky club of people who have no conception of the perils and problems he experienced, and it's pretty moving to be able to hear his thoughts on how to think about them millenia later.

Anyway he had this one piece of advice from [the Encheiridion](https://books.google.de/books?id=johjDwAAQBAJ&pg=PT35&lpg=PT35&dq=%22Well,+what+is+the+price+for+heads+of+lettuce?%22&source=bl&ots=H85whzW9QV&sig=ACfU3U0BdFyJAeU3uqQ5ki6Rt1U0gQjAFg&hl=en&sa=X&ved=2ahUKEwjukLq9u775AhXOMewKHSPlD4IQ6AF6BAgEEAM#v=onepage&q=%22Well%2C%20what%20is%20the%20price%20for%20heads%20of%20lettuce%3F%22&f=false) ("The Handbook") that I think of a lot when I look at our industry.  Epictetus is talking about how people jockey to do favors for the powerful, and the way it helps them rise in social or academic status, and that you, philosopher he is addressing, also want this status, but without all the ass-kissing, because it's immoral. And he says (slightly modernized from W.A. Oldfather translation):

"Well, what is the price of a head of lettuce? A dollar perhaps. If, then, somebody gives up his dollar and gets his head of lettuce, while you do not give your dollar, and do not get it, do not imagine that you are worse off than the man who gets his lettuce. For as he has his head of lettuce, so you have your dollar which you have not given away."

There are a few points here. First of all, if you really *believe* this kind of status-seeking is immoral, then you haven't lost anything when you've held onto your dollar and chosen not to buy the lettuce, and it's important to notice that – it can be difficult to notice because there are a lot of signals from our society which say that status is more important than values. You still have your morality, which should have a very high value to you, so that is actually a good deal. If it still feels like a bad deal, you may have a different opinion of the morality of the situation, or morality in general, than you think you do, or (more likely, in my experience) you may have substituted someone else's opinion for your own.

Second, IMO Epictetus is making a little joke by comparing status and morality to lettuce, and the amount of money that lettuce costs, respectively.

Third, there's an unspoken implication that either can still be chosen differently. Exchanging a dollar for lettuce is easier than exchanging lettuce for a dollar, granted – that is part of his point. But, you could sell your lettuce for a dollar or return it for your original dollar. You could spend your dollar on lettuce. Because of this, his main point is that you shouldn't complain, primarily if you have a dollar but no lettuce, which he thought was the better deal, but perhaps also if you have a lettuce and no dollar. Change the deal to the one you can live with, or accept your decision, but don't lose your limited time envying people who have something you decided that you don't want.

In the context of tech, I often think about this advice, not in terms of social status, which isn't a giant influence in the life of software developers (although it is, of course, in the mix), but more in terms of the question of what problems we choose to use our skills to solve and for what compensations, and how we feel about our choices.