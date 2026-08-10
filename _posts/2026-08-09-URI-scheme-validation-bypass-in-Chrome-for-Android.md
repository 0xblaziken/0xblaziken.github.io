---
layout: post
title: URI scheme validation bypass in Chrome for Android [CVE-2026-7941]
date: 2026-08-09 20:48 +0530
---
---
## Intent handling
 Chrome handles Url intents primarily in `ChromeTabbedActivity`.

 [initializeState](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;bpv=1;bpt=1;l=2360?gsn=initializeState&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Djava%3Fpath%3Dorg.chromium.chrome.browser.ChromeTabbedActivity%239573640f14fc1ebff194dd740168b8b7707e0d7d9feb11e433bd4bd9b75efb38) method and [onNewIntentWithNative](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;bpv=1;bpt=1;l=1937?gsn=onNewIntentWithNative&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Djava%3Fpath%3Dorg.chromium.chrome.browser.ChromeTabbedActivity%235218d9e34b197c1356334bf45f0dd562f8d3dd5835a24fdb23c534d3b5d1d6f4) handles intents at two stages, the former gets executed during the restoration of the tab state or during the fresh launch of the activity.
 
When there are any subsequent intents followed by launch it gets handled in `onNewIntentWithNative` method.

It processes three types of intents for opening urls:
1. Normal intent action [ACTION_VIEW](https://developer.android.com/guide/components/intents-common#Browser) containing single url.
2. [Multi tab](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/tabmodel/android/java/src/org/chromium/chrome/browser/tabmodel/MultiTabMetadata.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=22) intent for multiple url handling.
3. [Group Drag and drop](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/tabmodel/android/java/src/org/chromium/chrome/browser/tabmodel/TabGroupMetadata.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=27) intent.

## Scheme validation
>Before url gets loaded it goes through several checks in [shouldIgnoreIntentUrl](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/IntentHandler.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=1071).

>[isUnsafeExternalScheme](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ExternalIntentUrlChecker.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=76) checks if the lowercase representation of the url has **javascript:** or **jar:** or **googlechrome:** as scheme.

In [shouldIgnoreIntent](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/IntentHandler.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=1013) intents are checked in a order:

If "Multi tab" intent is present then it checks it and returns early, else if "Drag and drop" intent is present it checks it and returns, else it checks the single url intent.

## Processing
Ultimately, if `shouldIgnoreIntent` returns false, then processing of the intent happens at [maybeHandleUrlIntent](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=2036) depending on which type of intent they are.
This is done through checking if certain extras are present in the intent.

However the order of intent processing does not match that with the validation:
```java
   private boolean maybeHandleUrlIntent(Intent intent) {
        @Nullable TabGroupMetadata tabGroupMetadata = IntentHandler.getTabGroupMetadata(intent);
        if (tabGroupMetadata != null) {
            return maybeHandleGroupUrlsIntent(intent, tabGroupMetadata);
        }
        if (intent.hasExtra(IntentHandler.EXTRA_MULTI_TAB_REPARENTING_METADATA)) {
            return maybeHandleMultipleUrlIntent(intent);
        }
        return maybeHandleSingleUrlIntent(intent);
    }
```
During validation, "Multi tab" intent is validated first, but in the above code "Group drag and drop" is processed first.
This leads to bypass of security validation, whereas an attacker can pass both "Multi tab" and "Drag and drop" intent with valid urls; "Multi tab" intent would contain valid, harmless url, but "Drag and drop" would contain `javascript:` URI.

Since "Multi tab" intent is validated first, the check will pass, but during processing "Drag and Drop" intent is used for processing which would contain malicious `javascript:` URI.

This combined with an extra `com.android.browser.application_id` set to `com.android.chrome` will make chrome load the `javascript:` URI in current loaded tab by entering this switch [case](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=2911).


{% include embed/video.html src='/assets/videos/7941.mp4' type='video/mp4' title='Demo' %}

