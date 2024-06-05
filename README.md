### Introduction

Infobip RTC extensions is a Android library which provides extended functionality to Infobip RTC SDK.

Currently available functionalities are:

- video filter implementations

Here you will find an overview, and a quick guide on how to include and use these extensions in your application.
There is also in-depth reference documentation available.

## Getting the library

Obtaining the Infobip RTC Extensions library is straightforward via Gradle dependency, conveniently accessible from the
Maven Central repository. Simply integrate the following snippet into your `build.gradle` file:

```groovy
dependencies {
    implementation('com.infobip:infobip-rtc-extensions:+@aar') {
        transitive = true
    }
}
```

> **IMPORTANT NOTE**: Given the potential for diverse extensions within this library, we strongly recommend enabling
> [resource shrinking](https://developer.android.com/build/shrink-code#shrink-resources) within your application. This
> ensures optimal app size management.

## Video filters

The Infobip RTC supports user-defined video filters capable of manipulating outgoing video streams during calls. The
library provides an extensive implementation of commonly used video filters, making configuration easier and enabling
seamless integration.

Currently available implementations are:

- [`RTCVideoFilter`](#rtc-video-filter)

<a name="rtc-video-filter"></a>

### RTCVideoFilter

This filter allows users to modify their background during video calls.

Supported video filter modes include:

- Virtual background
  ([`RTCVideoFilterMode.VIRTUAL_BACKGROUND`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilterMode#virtual-background)) -
  Users can set a custom image to be displayed as their background
- Background blur
  ([`RTCVideoFilterMode.BACKGROUND_BLUR`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilterMode#background-blur)) -
  Users can blur their background.
- Face Track
  ([`RTCVideoFilterMode.FACE_TRACK`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilterMode#face-track)) -
  Automatically adjusts the video to keep the user's face centered and properly framed within the view.
- None ([`RTCVideoFilterMode.NONE`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilterMode#none)) -
  No video filtering is applied; video frames are passed through unchanged. This option is recommended over repeatedly
  reallocating video filter resources to avoid visible disruptions.

To utilize this feature, begin by creating an instance of
the [`RTCVideoFilter`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilter)
object. The constructor
accepts [`RTCVideoFilterOptions`](https://github.com/infobip/infobip-rtc-extensions-android/wiki/RTCVideoFilterOptions)
for customization.

```java
// Retrieve an image from a URL and decode it into a Bitmap
URL url = new URL("https://images.unsplash.com/photo-1558882224-dda166733046");
Bitmap image = BitmapFactory.decodeStream(url.openConnection().getInputStream());

// Create a RTCVideoFilterOptions object and set the virtual background image
RTCVideoFilterOptions options = RTCVideoFilterOptions.builder()
        .mode(RTCVideoFilterMode.VIRTUAL_BACKGROUND)
        .virtualBackground(image)
        .build();

// Create the RTCVideoFilter object with created options
RTCVideoFilter videoFilter = new RTCVideoFilter(options);
```

For optimal performance, it's recommended to avoid reallocating video filter instances solely for mode changes. Instead,
pass the new options directly to the existing video filter instance. This approach minimizes resource overhead and
enhances overall efficiency.

```java
// Create a RTCVideoFilterOptions object with default values
RTCVideoFilterOptions options = RTCVideoFilterOptions.builder().build();

// Set created options on existing video filter object
videoFilter.setOptions(options);
``` 

### Applying the video filter

Once you've created the video filter, you can utilize it during calls.

You can set it beforehand when initiating a
new [`ApplicationCall`](https://github.com/infobip/infobip-rtc-android/wiki/ApplicationCall)
using [`VideoOptions`](https://github.com/infobip/infobip-rtc-android/wiki/VideoOptions) object within
the [`ApplicationCallOptions`](https://github.com/infobip/infobip-rtc-android/wiki/ApplicationCallOptions) object:

```java
// Obtain authentication token and get instance of InfobipRTC
String token = obtainToken();
InfobipRTC infobipRTC = InfobipRTC.getInstance();

// Create a video application call with configured video options to use created video filter object
CallApplicationRequest callApplicationRequest = new CallApplicationRequest(token, getApplicationContext(), "45g2gql9ay4a2blu55uk1628", new DefaultApplicationCallEventListener());
VideoOptions videoOptions = VideoOptions.builder().videoFilter(videoFilter).build();
ApplicationCallOptions applicationCallOptions = ApplicationCallOptions.builder().video(true).videoOptios(videoOptions).build();
ApplicationCall applicationCall = infobipRTC.callApplication(callApplicationRequest, applicationCallOptions);
```

Alternatively, you can apply the filter to the
existing [`ApplicationCall`](https://github.com/infobip/infobip-rtc-android/wiki/ApplicationCall) using the
[`setVideoFilter`](https://github.com/infobip/infobip-rtc-android/wiki/ApplicationCall#set-video-filter) method:

```java
// Retrieve the active application call and set the video filter to created video filter
ApplicationCall applicationCall = InfobipRTC.getInstance().getActiveApplicationCall();
applicationCall.setVideoFilter(videoFilter);
```

### Implementing your own

If you wish to provide your own implementation of video filters, you need to implement the
[`VideoFilter`](https://github.com/infobip/infobip-rtc-android/wiki/VideoFilter) interface. 
For example, a trivial video filter which draws a red diagonal line would look like this:

```java
class RedLineFilter implements VideoFilter {
    private VideoFilterManager videoFilterManager; 
    
    @Override
    public void applyFilter(Bitmap bitmap, int rotation, long timestampNs) {
        // Create a mutable copy of the bitmap to apply the filter
        Bitmap filteredFrame = bitmap.copy(Bitmap.Config.ARGB_8888, true);

        // Apply the red line filter diagonally from the top-left corner
        for (int i = 0; i < Math.min(filteredFrame.getWidth(), filteredFrame.getHeight()); ++i) {
            filteredFrame.setPixel(i, i, Color.RED);
        }

        // Provide the filtered frame. This call can be done from another thread in case your
        // filter logic is asynchronous, but keep in mind that in that case the source bitmap
        // has to be copied if it's going to be read after `applyFilter` returns!
        videoFilterManager.notifyFrameProcessed(filteredFrame, rotation, timestampNs);
    }

    @Override
    public void start(int width, int height, int sourceFps, Context context, VideoFilterManager videoFilterManager) throws VideoFilterException {
      this.videoFilterManager = videoFilterManager;
    }

    @Override
    public void stop() {
    }
}
```
