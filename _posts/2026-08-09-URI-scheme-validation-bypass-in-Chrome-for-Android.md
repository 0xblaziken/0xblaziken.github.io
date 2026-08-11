---
layout: post
title: URI scheme validation bypass in Chrome for Android [CVE-2026-7941]
date: 2026-08-09 20:48 +0530
---
---
## Intent handling
 Chrome handles URL intents primarily in `ChromeTabbedActivity`.

 [initializeState](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;bpv=1;bpt=1;l=2360?gsn=initializeState&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Djava%3Fpath%3Dorg.chromium.chrome.browser.ChromeTabbedActivity%239573640f14fc1ebff194dd740168b8b7707e0d7d9feb11e433bd4bd9b75efb38) method and [onNewIntentWithNative](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;bpv=1;bpt=1;l=1937?gsn=onNewIntentWithNative&gs=KYTHE%3A%2F%2Fkythe%3A%2F%2Fchromium.googlesource.com%2Fcodesearch%2Fchromium%2Fsrc%2F%2Fmain%3Flang%3Djava%3Fpath%3Dorg.chromium.chrome.browser.ChromeTabbedActivity%235218d9e34b197c1356334bf45f0dd562f8d3dd5835a24fdb23c534d3b5d1d6f4) handles intents at two stages: 
 The former gets executed during the restoration of the tab state or during the fresh launch of the activity.
 
When there are any subsequent intents followed by launch, it gets handled in `onNewIntentWithNative` method.

It processes three types of intents for opening URLs:
1. Normal intent action, [ACTION_VIEW](https://developer.android.com/guide/components/intents-common#Browser) containing a single URL.
2. [multi-tab](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/tabmodel/android/java/src/org/chromium/chrome/browser/tabmodel/MultiTabMetadata.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=22) intent for multiple URL handling.
3. [Group drag and drop](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/tabmodel/android/java/src/org/chromium/chrome/browser/tabmodel/TabGroupMetadata.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=27) intent.

## Scheme validation
>Before the URL gets loaded, it goes through several checks in [shouldIgnoreIntentUrl](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/IntentHandler.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=1071).

>`isInvalidScheme` checks if the lowercase representation of the URL has **javascript:** or **jar:** or **googlechrome:** as a scheme.
```java
 private static boolean isInvalidScheme(@Nullable String scheme) {
        return scheme != null
                && (scheme.toLowerCase(Locale.US).equals(UrlConstants.JAVASCRIPT_SCHEME)
                        || scheme.toLowerCase(Locale.US).equals(UrlConstants.JAR_SCHEME));
    }
```

In [shouldIgnoreIntent](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/IntentHandler.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=1013), intents are checked in an order:

If the "multi-tab" intent is present, then it checks it and returns early; else, if "drag and drop" intent is present, it checks it and returns; else, it checks the single URL intent.
```java
 public static boolean shouldIgnoreIntent(
            Intent intent, @Nullable Context context, boolean isCustomTab) {
        // Although not documented to, many/most methods that retrieve values from an Intent may
        // throw. Because we can't control what packages might send to us, we should catch any
        // Throwable and then fail closed (safe). This is ugly, but resolves top crashers in the
        // wild.
        try {
            // If the intent contains a list of tabs to reparent, it's a valid intent from Chrome.
            @Nullable MultiTabMetadata multiTabMetadata = getMultiTabMetadata(intent);
            if (multiTabMetadata != null) {
                // Exit early if the incognito intent is not allowed.
                if (IntentUtils.safeGetBooleanExtra(intent, EXTRA_OPEN_NEW_INCOGNITO_TAB, false)
                        && !isAllowedIncognitoIntent(
                                wasIntentSenderChrome(intent), isCustomTab, intent)) {
                    return true;
                }
                ArrayList<Integer> tabIds = multiTabMetadata.tabIds;
                ArrayList<String> urls = multiTabMetadata.urls;

                if (urls == null || tabIds == null || urls.size() != tabIds.size()) {
                    assert false : "Urls and tabIds size are mismatched or empty.";
                    return true;
                }

                for (int i = urls.size() - 1; i >= 0; i--) {
                    if (shouldIgnoreIntentUrl(intent, context, urls.get(i), isCustomTab)) {
                        urls.remove(i);
                        tabIds.remove(i);
                    }
                }
                return urls.isEmpty();
            }
            // Ignore all invalid URLs, regardless of what the intent was.
            @Nullable TabGroupMetadata tabGroupMetadata = IntentHandler.getTabGroupMetadata(intent);
            if (tabGroupMetadata != null) {
                // Exit early if the incognito intent is not allowed.
                if (tabGroupMetadata.isIncognito
                        && !isAllowedIncognitoIntent(
                                wasIntentSenderChrome(intent), isCustomTab, intent)) {
                    return true;
                }

                // Check url validity and remove invalid urls if needed.
                List<Entry<Integer, String>> tabIdsToUrls = tabGroupMetadata.tabIdsToUrls;
                Iterator<Entry<Integer, String>> iterator = tabIdsToUrls.iterator();
                while (iterator.hasNext()) {
                    Map.Entry<Integer, String> entry = iterator.next();
                    String url = entry.getValue();
                    if (shouldIgnoreIntentUrl(intent, context, url, isCustomTab)) {
                        iterator.remove();
                    }
                }
                // TODO(crbug.com/384979079) Add metrics for invalid url and ignored intent during
                // group drag drop.
                return tabIdsToUrls.size() == 0;
            } else {
                return shouldIgnoreIntentUrl(
                        intent, context, getUrlFromIntent(intent), isCustomTab);
            }
        } catch (Throwable t) {
            return true;
        }
```

## Processing
Ultimately, if `shouldIgnoreIntent` returns false, then processing of the intent happens at [maybeHandleUrlIntent](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=2036) depending on which type of intent they are.
This is done through checking if certain extras are present in the intent.

However, the order of intent processing does not match that with the validation:
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
During validation, the "multi-tab" intent is validated first, but in the above code "Group drag and drop" is processed first.
This leads to a bypass of security validation, whereas an attacker can pass both "multi-tab" and "drag and drop" intent with valid URLs; "multi-tab" intent would contain valid, harmless URLs, but "drag and drop" would contain `javascript:` URI.

Since the "multi-tab" intent is validated first, the check will pass, but during processing the "drag and drop" intent is used for processing, which would contain malicious `javascript:` URI.

This combined with an extra `com.android.browser.application_id` set to `com.android.chrome` will make Chrome load the `javascript:` URI in the currently loaded tab by entering this switch [case](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/ChromeTabbedActivity.java;drc=c8738dd61c7284dfa1e34c973c7b82b2c0d99c30;l=2911).


{% include embed/video.html src='/assets/videos/7941.mp4' type='video/mp4' title='Demo' %}

