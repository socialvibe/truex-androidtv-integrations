![true[X] logo](media/truex.png)


# TruexAdRenderer Android/Fire TV Documentation

Version 2.1.0

## Contents

* [Overview](#overview)
* [Product Flows](#product-flows)
* [How to use TruexAdRenderer](#how-to-use-truexadrenderer)
    * [When to show true[X]](#when-to-show-truex)
    * [Handling Events from TruexAdRenderer](#handling-events-from-truexadrenderer)
        * [Terminal Events](#terminal-events)
    * [Handling Ad Elimination](#handling-ad-elimination)
* [TruexAdRenderer Android/Fire TV API](#truexadrenderer-androidfire-tv-api)
    * [Adding `TruexAdRenderer` to Your Project](#adding-truexadrenderer-to-your-project)
        * [Gradle Example](#gradle-example)
    * [`TruexAdRenderer` Methods](#truexadrenderer-methods)
        * [`init`](#init)
        * [`start`](#start)
        * [`stop`](#stop)
        * [`pause`](#pause)
        * [`resume`](#resume)
    * [`TruexAdRenderer` Output Events -- Main Flow](#truexadrenderer-output-events----main-flow)
        * [`AD_FETCH_COMPLETED`](#ad_fetch_completed)
        * [`AD_STARTED`](#ad_started)
        * [`AD_COMPLETED`](#ad_completed)
        * [`AD_ERROR`](#ad_error)
        * [`NO_ADS_AVAILABLE`](#no_ads_available)
        * [`AD_FREE_POD`](#ad_free_pod)
        * [`POPUP_WEBSITE`](#popup_website)
        * [`USER_CANCEL_STREAM`](#user_cancel_stream)
    * [`TruexAdRenderer` Output Events -- Informative](#truexadrenderer-output-events----informative)
        * [`OPT_IN`](#opt_in)
        * [`OPT_OUT`](#opt_out)
        * [`SKIP_CARD_SHOWN`](#skip_card_shown)
        * [`USER_CANCEL`](#user_cancel)
        * [`VIDEO_EVENT`](#video_event)

## Overview

In order to support interactive ads on Android TV and Fire TV, true[X] has created a renderer library that can renderer true[X] ads natively, which interfaces with a hosting app, as well as its existing ad server and content delivery mechanism (e.g. SSAI).

With this library, the host player app can defer to the TruexAdRenderer when it is required to display a true[X] ad.

For simplicity, publisher implemented code will be referred to as "app code" while true[X] implemented code will be referred to as "renderer code".

true[X] will provide a Java `TruexAdRenderer` library that can be loaded into the app. This library will offer a class, `TruexAdRenderer`, that will need to be instantiated, initialized and given certain commands (described below in [TruexAdRenderer Methods](#truexadrenderer-methods)) by the app code. It will also contain a class of shared constants, `TruexAdRendererConstants`.

At this point, the renderer code will take on the responsibility of requesting ads from true[X] server, creating the native UI for the true[X] choice card and interactive ad unit, as well as communicating events to the app code when action is required.

The app code will still need to parse out the SSAI ad response, detect when a true[X] ad is supposed to display, pause the stream, instantiate `TruexAdRenderer` and handle any events emitted by the renderer code. It will also need to call pause, resume and stop methods on the `TruexAdRenderer` when certain external events happen, like if the app is backgrounded or if the user has requested to exit the requested stream via back buttons.

It will also need to handle skipping ads in the current ad pod, if it is notified to do so.


## Product Flows

There are two distinct product flows supported by `TruexAdRenderer`: Sponsored Stream (full-stream ad-replacement) and Sponsored Ad Break (mid-roll ad-replacement).

In a Sponsored Ad Break flow, once the user hits a mid-roll break with a true[X] tag flighted, they will be shown a "choice-card" offering them the choice between watching a normal set of video ads or a fully interactive true[X] ad:

![choice card](media/choice_card.png)

_**Fig. A** example true[X] mid-roll choice card_

If the user opts for a normal ad break, or if the user does not make a selection before the countdown timer expires, the true[X] UI will close and playback of normal video ads can continue as usual.

If the user opts to interact with true[X], an interactive ad unit will be shown to the user:

![ad](media/ad.png)

_**Fig. B** example true[X] interactive ad unit_

The requirement for the user to "complete" this ad is for them to spend at least the allotted time on the unit and for at least one interaction (e.g. navigating anywhere through the ad).

![true attention timer example](media/true-attention-timer-example.png)

_**Fig. C** example true[X] attention timer_

Once the user fulfills both requirements, a "Watch Your Show" button will appear in the bottom right, which the user can select to exit the true[X] ad. Having completed a true[X] ad, the user will be returned directly to content, skipping the remaining ads in the current ad pod.

The Sponsored Stream flow is quite similar. In this scenario, a user will be shown a choice-card in the pre-roll:

![choice card](media/choice_card.png)

_**Fig. D** example true[X] pre-roll choice card (full-stream replacement)_

Similarly, if the user opts-in and completes the true[X] ad, they will be skipped over the remainder of the pre-roll ad break. However, every subsequent mid-roll break in the current stream will also be skipped over. In this case instead of the regular pod of video ads, the user will be shown a "hero card" (also known as a "skip card"):

![skip card](media/skip_card.png)

_**Fig. E** example true[X] mid-roll skip card_

This messaging will be displayed to the user for several seconds, after which they will be returned directly to content.


## How to use TruexAdRenderer

### When to show true[X]

Upon receiving an ad schedule from your SSAI service, you should be able to detect whether or not true[X] is returned in any of the pods. true[X] ads should have `apiFramework` set to `VPAID` or `truex`.

SSAI vendors differ in the way they convey information back about ad schedules to clients. Certain vendors such as Verizon / Uplynk expose API’s which return details about the ad schedule in a JSON object. For other vendors, for instance Google DAI, the true[X] payload will be encapsulated as part of a companion payload on the returned VAST ad. Please work with your true[X] point of contact if you have difficulty identifying the right approach to detecting the true[X] placeholder, which will be the trigger point for the ad experience.

Once the player reaches a true[X] placeholder, it should pause, instantiate the `TruexAdRenderer` and immediately call [`init`](#init) followed by [`start`](#start).

Alternatively, you can instantiate and `init` the `TruexAdRenderer` in preparation for an upcoming placeholder. This will give the `TruexAdRenderer` more time to complete its initial ad request, and will help streamline true[X] load time and minimize wait time for your users. Once the player reaches a placeholder, it can then call `start` to notify the renderer that it can display the unit to the user.


### Handling Events from TruexAdRenderer

Once `start` has been called on the renderer, it will start to emit events (see [`TruexAdRenderer` Output Events -- Main Flow](#truexadrenderer-output-events----main-flow) and [`TruexAdRenderer` Output Events -- Informative](#truexadrenderer-output-events----informative)).

One of the first events you will receive is [`AD_STARTED`](#ad_started). This notifies the app that the renderer has received an ad for the user and has started to show the unit to the user. The app does not need to do anything in response, however it can use this event to facilitate a timeout. If an `AD_STARTED` event has not fired within a certain amount of time after calling `start`, the app can call `stop` on the renderer and proceed to normal video ads.

At this point, the app must listen for the renderer's [terminal events](#terminal-events), while paying special attention to the [`AD_FREE_POD`](#ad_free_pod) event. A *terminal event* signifies that the renderer is done with its activities and the app may now resume playback of its stream. The `AD_FREE_POD` event signifies that the user has earned a credit with true[X] and all linear video ads remaining in the current pod should be skipped. If the `AD_FREE_POD` event did not fire before a terminal event is emitted, the app should resume playback without skipping any ads, so the user receives a normal video ad payload.

#### Terminal Events

The *terminal events* are:
* [`AD_COMPLETED`](#ad_completed): the user has exited the true[X] ad
* [`USER_CANCEL_STREAM`](#user_cancel_stream): the user intends to exit the current stream entirely
* [`NO_ADS_AVAILABLE`](#no_ads_available): there were no true[X] ads available to the user
* [`AD_ERROR`](#ad_error): the renderer encountered an unrecoverable error

It's important to note that the player should not immediately resume playback once receiving the `AD_FREE_POD` event -- rather, it should note that it was fired and continue to wait for a terminal event.


### Handling Ad Elimination

Skipping video ads is completely the responsibility of the app code. The SSAI API should provide enough information for the app to determine where the current pod end-point is, and the app, when appropriate, should fast-forward directly to this point when resuming playback.


## TruexAdRenderer Android/Fire TV API

This is an outline of public `TruexAdRenderer` methods and events. Methods are called on the `TruexAdRenderer` instance, while output events are triggered by calls to listeners registered via `TruexAdRenderer.addEventListener`.


### Adding `TruexAdRenderer` to Your Project

The easiest way to add `TruexAdRenderer` is via Maven. The renderer will be maintained and distributed on Github.

1. Add the Maven repository to your project:
    ```
    https://ctv.truex.com/android/maven
    ```
1. Add the dependency:
    * group ID: `com.truex`
    * artifact ID: `TruexAdRenderer-tv`
    * version: `2.1.2`

#### Gradle Example

```groovy
repositories {
    maven {
        url "https://ctv.truex.com/android/maven"
    }
}

dependencies {
    implementation 'com.truex:TruexAdRenderer-tv:2.1.2'
}
```


### TruexAdRenderer Methods

#### `init`

```java
    public void init(JSONObject adParameters, String slotType)
```

This method should be called by the app code in order to initialize the `TruexAdRenderer`. The renderer will parse out the `adParameters` and `slotType` passed to it and make a request to the true[X] ad server to see what ads are available.

You may initialize `TruexAdRenderer` early (a few seconds before the next pod even starts) in order to give it extra time to make the ad request. The renderer will output an `AD_FETCH_COMPLETED` event at completion of this ad request. This event can be used to facilitate the implementation of a timeout or loading indicator, and when to make the call to `start`.

The parameters for this method call are:

* `adParameters`: AdParameters as returned by SSAI. In the example of Uplynk, this would correspond to `response.ads.breaks[0].ads[0].adParameters`
* `slotType`: the type of the current ad pod, `PREROLL` or `MIDROLL`


#### `start`

```java
    public void start(final ViewGroup baseView)
```

This method should be called by the app code when the app is ready to display the true[X] unit to the user. This can be called anytime after the unit is initialized. 

The app should have as much extraneous UI hidden as possible, including player controls, status bars and soft buttons/keyboards.

After calling `start`, the app code should wait for a [terminal event](#terminal-events) before taking any more action, while keeping track of whether or not the [`AD_FREE_POD`](#ad_free_pod) event has fired.

In a non-error flow, the renderer will first wait for the ad request triggered in `init` to finish if it has not already. It will then display the true[X] unit to the user in a new `View` presented on the `baseView` and then fire the [`AD_STARTED`](#ad_started) event. `AD_FREE_POD` and other events may fire after this point, depending on the user's choices, followed by one of the [terminal events](#terminal-events).

The parameters for this method call are:

* `baseView`: The `ViewGroup` that `TruexAdRenderer` should present its `View` to. The `baseView` should cover the entire screen.


#### `stop`

```java
    public void stop()
```

The `stop` method is only called when the app needs the renderer to immediately stop and destroy all related resources. Examples include:
* the user backs out of the video stream to return to the normal app UI
* there was an unrelated error that requires immediate halt of the current ad unit
* the app code has reached a custom timeout waiting for either the [`AD_FETCH_COMPLETED`](#ad_fetch_completed) or [`AD_STARTED`](#ad_started) events

The `TruexAdRenderer` instance should not be used again after calling `stop` -- please remove all references to it afterwards.

In contrast to `pause`, there is no way to resume the ad after `stop` is called.


#### `pause`

```java
    public void pause()
```

`pause` is required whenever the hosted app needs to be paused, either for the app life-cycle events, or android broadcast events like HDMI state change, [see BroadcastReceiver example](https://github.com/socialvibe/Sheppard/blob/master/Sheppard/src/main/java/com/truex/sheppard/MainActivity.java#L289-L302). Pausing the true[X] unit will pause its videos, audios, and timers.


#### `resume`

```java
    public void resume()
```

`resume` should be called when the app is ready for a previously paused true[X] unit to resume.


### `TruexAdRenderer` Output Events -- Main Flow

The following events signal the main flow of the `TruexAdRenderer` and may require action by the host app:


#### `AD_FETCH_COMPLETED`

```java
    IEventHandler adFetchCompletedHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.AD_FETCH_COMPLETED, adFetchCompletedHandler);
```

This event fires in response to the `init` method when the true[X] ad request has successfully completed and the ad is ready to be presented. The host app may use this event to facilitate a loading screen for pre-rolls, or to facilitate an ad request timeout for mid-rolls.

For example: `init` is called for the pre-roll slot, and the app code shows a loading indicator while waiting for `AD_FETCH_COMPLETED`. Then, either the event is received (and the app can call `start`) or a specific timeout is reached (and the app can call `stop` and resume playback of normal video ads).

Another example: `init` is called well before a mid-roll slot to give the renderer a chance to preload its ad. If `AD_FETCH_COMPLETED` is received before the mid-roll slot is encountered, then the app can call `start` to immediately present the true[X] ad. If not, the app can wait for a specific timeout (if not already reached, in case the user has seeked to activate the mid-roll) before calling `stop` on the renderer and resuming playback with normal video ads.


#### `AD_STARTED`

```java
    IEventHandler adStartedHandler = (Map<String, ?> data) -> {
        String campaignName = (String) data.get("campaignName");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.AD_STARTED, adStartedHandler);
```

This event will fire in response to the `start` method when the true[X] UI is ready and has been added to the view hierarchy.

The parameters for this event are:

* `campaignName`: The name of the ad campaign available to the user (e.g. "*Storebought Coffee - Sample Video Ad #1 - Q1 2017*")


#### `AD_COMPLETED`

```java
    IEventHandler adCompletedHandler = (Map<String, ?> data) -> {
        int timeSpent = (Integer) data.get("timeSpent");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.AD_COMPLETED, adCompletedHandler);
```

This is a [terminal event](#terminal-events). This event will fire when the true[X] unit is finished with its activities -- at this point, the app should resume playback and remove all references to the `TruexAdRenderer` instance.

Here are some examples where `AD_COMPLETED` will fire:

* The user opts for normal video ads (not true[X])
* The choice card countdown runs out
* The user completes the true[X] ad unit
* After a "skip card" has been shown to a user for its duration

The parameters for this event are:

* `timeSpent`: The amount of time (in seconds) the user spent on the true[X] unit


#### `AD_ERROR`

```java
    IEventHandler adErrorHandler = (Map<String, ?> data) -> {
        String errorMessage = (String) data.get("errorMessage");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.AD_ERROR, adErrorHandler);
```

This is a [terminal event](#terminal-events). This event will fire when the true[X] unit has encountered an unrecoverable error. The app code should handle this the same way as an `AD_COMPLETED` event -- resume playback and remove all references to the `TruexAdRenderer` instance.

The parameters for this event are:

* `errorMessage`: A description of the cause of the error.


#### `NO_ADS_AVAILABLE`

```java
    IEventHandler noAdsAvailableHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.NO_ADS_AVAILABLE, noAdsAvailableHandler);
```

This is a [terminal event](#terminal-events). This event will fire when the true[X] unit has determined it has no ads available to show the current user. The app code should handle this the same way as an `AD_COMPLETED` event -- resume playback and remove all references to the `TruexAdRenderer` instance.


#### `AD_FREE_POD`

```java
    IEventHandler adFreePodHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.AD_FREE_POD, adFreePodHandler);
```

This event will fire when the user has earned a credit with true[X]. The app code should notate that this event has fired, but should not take any further action. Upon receiving a [terminal event](#terminal-events), if `AD_FREE_POD` was fired, the app should skip all remaining ads in the current slot. If it was not fired, the app should resume playback without skipping any ads, so the user receives a normal video ad payload.


#### `POPUP_WEBSITE`

```java
    IEventHandler popupWebsiteHandler = (Map<String, ?> data) -> {
        String url = (String) data.get("url");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.POPUP_WEBSITE, popupWebsiteHandler);
```

This event will fire when a popup view is required from the app. The app should call `pause` on the renderer and display the requested URL to the user. Once the user closes the popup, the app should call `resume` on the renderer.

The parameters for this event are:

* `url`: The URL to open in the popup.


#### `USER_CANCEL_STREAM`

```java
    IEventHandler userCancelStreamHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.USER_CANCEL_STREAM, userCancelStreamHandler);
```

This is a [terminal event](#terminal-events), and is optional.

Adding a listener for this event will enable the renderer to fire this event when the user intends to exit the stream entirely. The app, at this point, should treat this the same way it would handle any other "exit" action from within the stream -- in most cases this will result in the user being returned to an episode/series detail page.

If the app code does not add a listener for this event, then the renderer will emit an [`AD_COMPLETED`](#ad_completed) event instead.


### `TruexAdRenderer` Output Events -- Informative

All following events are used mostly for tracking purposes -- no action is generally required.


#### `OPT_IN`

```java
    IEventHandler optInHandler = (Map<String, ?> data) -> {
        String campaignName = (String) data.get("campaignName");
        int adID = (Integer) data.get("adID");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.OPT_IN, optInHandler);
```

This event will fire if the user selects to interact with the true[X] interactive ad.

Note that this event may be fired multiple times if a user opts in to the true[X] interactive ad and subsequently backs out.

The parameters for this event are:

* `campaignName`: The name of the ad campaign.
* `adID`: The ad ID.


#### `OPT_OUT`

```java
    IEventHandler optOutHandler = (Map<String, ?> data) -> {
        boolean userInitiated = (Boolean) data.get("userInitiated");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.OPT_OUT, optOutHandler);
```

This event will fire if the user opts for a normal video ad experience.

The parameters for this event are:

* `userInitiated`: This will be set to `true` if this was actively selected by the user, `false` if the user simply allowed the choice card countdown to expire.


#### `SKIP_CARD_SHOWN`

```java
    IEventHandler skipCardShownHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.SKIP_CARD_SHOWN, skipCardShownHandler);
```

This event will fire anytime a "skip card" is shown to a user as a result of completing a true[X] Sponsored Stream interactive ad in an earlier pre-roll.


#### `USER_CANCEL`

```java
    IEventHandler userCancelHandler = (Map<String, ?> data) -> {
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.USER_CANCEL, userCancelHandler);
```

This event will fire when a user backs out of the true[X] interactive ad unit after having opted in. This would be achieved by tapping the "Yes" link to the "Are you sure you want to go back and choose a different ad experience" prompt inside the true[X] interactive ad. The user will be subsequently taken back to the Choice Card (with the countdown timer reset to full).

Note that after a `USER_CANCEL`, the user can opt-in and engage with an interactive ad again, so more `OPT_IN` or `OPT_OUT` events may then be fired.


#### `VIDEO_EVENT`

```java
    IEventHandler videoEventHandler = (Map<String, ?> data) -> {
        String type = (String) data.get("subType");
        String videoName = (String) data.get("videoName");
        String url = (String) data.get("url");
        // [...]
    };
    truexAdRenderer.addEventListener(TruexAdRendererConstants.VIDEO_EVENT, videoEventHandler);
```

A `VIDEO_EVENT` is emitted when noteworthy events occur in a video within a true[X] unit. These events are differentiated by their `subType` value (found in `TruexAdRendererConstants`) as follows:

* `VIDEO_STARTED`: A video has started
* `VIDEO_FIRST_QUARTILE`: A video has reached its first quartile
* `VIDEO_SECOND_QUARTILE`: A video has reached its second quartile
* `VIDEO_THIRD_QUARTILE`: A video has reached its third quartile
* `VIDEO_COMPLETED`: A video has reached completion

The parameters for this event are:

* `subType`: The type of the emitted video event.
* `videoName`: The name of the video that emitted the event, if available
* `url`: The source URL of the video that emitted the event, if available
