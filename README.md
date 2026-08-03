# Brokering FedCM via an on-device application

# TL;DR;

This document goes over an extension to FedCM to allow it to talk a local application as an intermediary to the Identity Provider (IdP).  Native application brokers are a class of native applications that run outside the scope of the web browser that can interact with and Identity Provider (IdP) to sign a user in.  This is useful in scenarios where a given application might be signed into an IdP and the in-browser web application wants to leverage this sign in state for browser scenarios.  It could also be used for web applications that want to leverage the sign in state of the operating system itself.

# Context

FedCM is a Web Platform API that allows the browser to intermediate Relying Parties requesting accounts from Identity Providers.

It does so by introducing a protocol that a cooperating relying party (a website you are trying to log in to) and identity provider (a website that you are already typically logged in) have to conform to, allowing the browser to mediate between the two.

More details about FedCM here:

[https://privacysandbox.google.com/cookies/fedcm](https://privacysandbox.google.com/cookies/fedcm)  
[https://developer.mozilla.org/en-US/docs/Web/API/FedCM\_API](https://developer.mozilla.org/en-US/docs/Web/API/FedCM_API) 

# The Problem

A key aspect of the FedCM protocol is that there are two endpoints that are exposed by the Identity Provider that require the user to be logged in to the IdP: the `accounts endpoint` and the `id assertion endpoint`.

The first is an endpoint that, given a session cookie, returns all of the accounts that the user is logged in to, which the browser uses to build a mediated account chooser. The second is an endpoint that, given a session cookie and a specific account, which the browser uses to generate a cryptographic token that allows the user to login to the website.

This works well on desktop, because, for the most part, the user is logged in to their identity providers on desktop in the browser (think, [google.com](http://google.com), [facebook.com](http://facebook.com), [twitter.com](http://twitter.com), [github.com](http://github.com), [microsoft.com](http://microsoft.com), [gmx.de](http://gmx.de), [web.de](http://web.de), etc), and so the user gets to reuse these accounts to login to other websites.

The problem is that this is less the case on mobile devices, and some set of desktop devices: users are largely logged in to native applications (e.g. the facebook native app, the GMX native email app, a company managed Authenticator app, etc.) rather than their web counterparts (e.g. [facebook.com](http://facebook.com) or [gmx.de](http://gmx.de) website).

For some IdPs, a meaningful part of their users are not seamlessly logged in to their Web Apps.

# The Proposal

The proposal being explored here is to introduce a new transport for FedCM, called FedCM-Over-Services, that the browser can use to talk to locally installed Apps as an alternative to HTTP, and vice versa (as a mechanism that the native app can use to talk to the browser).

In this proposal, the browser takes a FedCM request as it normally would, but now supports a new convention that allows a cooperating application to expose itself as a FedCM Identity Provider.

## Android specifics -- needs to be updated to include, iOS, Mac, Windows, etc.

The transport is based on Android’s Bound Services using the standard Messenger and Bundle messaging protocol (with request/reply message codes and JSON string payloads). It is used for authenticated requests that FedCM needs to make, namely the accounts endpoint and the id assertion endpoint.

It also works to let the IdP’s native app talk to the browser, for example for the Login Status API. 

All of the other uncredentialed requests are made through HTTP.

## The Accounts and Id Assertion Endpoint

The transport is used as a graceful fallback to the native application whenever the user is not logged in to the web app (in the unknown case, we first try to fetch with cookies and otherwise fallback to this new transport).

For enterprise scenarios, policy can be set in the browser to specify which type of transport to attempt first, or possibly to mandate a specific transport.

On the IdP side, the IdP is required to expose a Service (a) in their manifest file and (b) as a class that extends Service, and respond to requests for the accounts endpoint and the id assertion endpoint.

When the IdP receives a message, it uses its internal OS storage to get their session credentials and is responsible for making HTTP requests to their backends on their own (e.g. without requiring the IdP to keep the cookies in sync between the app and the browser).

The IdP is able to reliably determine who is calling the app and needs to make a determination if it should reply to it or not (e.g. it can hard code the list of browsers as trusted callers).

On the Browser side, the browser assumes that the IdP is complying to the convention, and degrades gracefully when they are not (e.g. invalid responses or app not installed), just like it handles HTTP errors (400s and 500s).

The browser uses app links and the various ways that origins are verified with package names to make sure it is talking to the right app.

For example, lets say a website, say [rp.com](http://rp.com), calls into the FedCM API to get accounts from [idp.com](http://idp.com): 

```javascript
const credential = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: "https://idp.com/fedcm.json",
    }]
  }
});
```

The browser would go through its usual FedCM flow and at some point figure out that the user is not logged in to [idp.com](http://idp.com).

The browser can then try to see if the user is logged in to the equivalent native application for [idp.com](http://idp.com).

### Android
The browser starts by using Verified-App-Links to make the translation between [idp.com](http://idp.com) and the Android verified package name “com.idp.app” which rely on a bi-directional declaration in a well-known file: (e.g. [https://example.com/.well-known/assetlinks.json](https://example.com/.well-known/assetlinks.json)) and the Android app:

[https://developer.android.com/training/app-links/verify-applinks](https://developer.android.com/training/app-links/verify-applinks) 

(See the security section for more information)

Once the browser knows what the package name is, it uses the agreed-upon ahead of time convention of FedCM-Over-Services.

First, at compilation time, the browser declares that it will query all apps that have exposed the “org.w3.FedCM” intent signature:

See for more information: [https://developer.android.com/training/package-visibility/declaring](https://developer.android.com/training/package-visibility/declaring) 

```xml
<!--
  NOTE: this isn't necessary for Clank specifically, because it already has a
  QUERY_ALL_PACKAGES permission (which is a superset of this declaration).
-->
<queries>
  <intent>
    <action android:name="org.w3.FedCM" />
  </intent>
</queries>
```

Then, at run time, creates an intent to the specific application it wants to talk to:

```java

   // Query all apps that expose a FedCM service:
   Intent intent = new Intent("org.w3.FedCM");
   List<ResolveInfo> services = getPackageManager().queryIntentServices(intent, 0);
   if (services.isEmpty()) {
     serviceResponse.setText("No FedCM Services found.");
     return;
   }
   // Select from the list of FedCM services which one we want to talk to
   // TODO: check that the package name matches the origin
   ResolveInfo serviceInfo = services.get(0);
   ComponentName name = new ComponentName(
     serviceInfo.serviceInfo.packageName, serviceInfo.serviceInfo.name);
   intent.setComponent(name);

   ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Messenger serviceMessenger = new Messenger(service);
            Message msg = Message.obtain(null, 1, 0, 0);
            Bundle requestBundle = new Bundle();
            requestBundle.putString("request", "{"accounts_url":"https://idp.example/fedcm/accounts"}");
            msg.setData(requestBundle);

            msg.replyTo = new Messenger(new Handler(Looper.getMainLooper()) {
              @Override
              public void handleMessage(Message msg) {
                String reply = msg.getData().getString("reply");
              }
           });
           serviceMessenger.send(msg);
           unbindService(serviceConnection);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
   };

   bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
```

The cooperating Identity Providers must have had, ahead of time, declared that it supports the agreed upon service in their Android manifest file:

```xml
<service android:name=".FedCMService" android:exported="true">
  <intent-filter>
    <action android:name="org.w3.FedCM" />
   </intent-filter>
</service>
```

Which then, at run time, gets the message from the browser:

```java

public class HelloService extends Service {

    private Messenger messenger;

    static class IncomingHandler extends Handler {
        private final Context applicationContext;

        IncomingHandler(Context context) {
            super(Looper.getMainLooper());
            applicationContext = context;
        }

        @Override
        public void handleMessage(Message msg) {
            String callingApp = applicationContext.getPackageManager().getNameForUid(msg.sendingUid);
            // Checks that the calling app is trustworthy
            // e.g. should probably use the browsers allow list
            if (!"com.example.client".equals(callingApp)) {
                Log.w("HelloService", "Unauthorized client: " + callingApp);
                return;
            }

            if (msg.what == 1) {
                Messenger replyTo = msg.replyTo;
                String requestJson = msg.getData().getString("request");
                String replyJson = "{"accounts":[{"id":"1234","name":"Alice","email":"alice@idp.example"}]}";
                Message replyMsg = Message.obtain(null, 2);
                Bundle bundle = new Bundle();
                bundle.putString("reply", replyJson);
                replyMsg.setData(bundle);
                try {
                    replyTo.send(replyMsg);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            } else {
                super.handleMessage(msg);
            }
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        messenger = new Messenger(new IncomingHandler(getApplicationContext()));
        return messenger.getBinder();
    }
}
```

When the native IdP Android Application replies back to the browser, it can return credentialed requests like the accounts endpoint and the id assertion endpoint, using its own internal storage to authenticate to its backend services.

Ta-da.

### iOS

Needs definition.  This is doable on enterprise managed devices through some of the enterprise management services on the platform, a consumer focused implementation would need some investigation.

### Mac

Needs definition. We do this today with Edge and Chrome plugins to tunnel into a native broker.

### Windows

Needs definition. We do this today with Edge and Chrome plugins to tunnel into a native broker.

## The Login Status API

The browser exposes itself as a Bound Service that conforms to a certain convention that accepts “SetLogin” requests from native IdPs.

```xml
<service
  android:name=".LoginStatusService"
  android:exported="true">
  <intent-filter>
    <action android:name="org.w3.FedCM.LOGIN_STATUS" />
  </intent-filter>
</service>
```

And the corresponding browser implementation:

```java
/** Android Bound Service accepting login status updates from external native IdP applications. */
@NullMarked
public class LoginStatusService extends Service {
    private static final String TAG = "LoginStatusSvc";

    public static class LoginStatusBinder extends Binder {
        public boolean setLoginStatus(String status, String origin) {
            // ... writes to browser storage ...
            return true;
        }
    }

    private final IBinder mBinder = new LoginStatusBinder();

    @Override
    public @Nullable IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

So, ahead of time, the browser expects that the Android native app would tell the browser when their users are logging in and out of their native apps.  

In Chromium's implementation, `LoginStatusService` is implemented as an exported Bound Service that accepts login status updates from native IdP applications.

## Continuation API via Native Applications

When an IdP requires multi-step authentication or explicit user interaction (such as accepting terms of service or selecting a profile), the IdP can return a `continue_on` URL during a FedCM account or token request. 

Instead of opening a browser tab or Custom Tab, FedCM can dispatch the continuation flow directly to the IdP's installed Android application.

### 1. Intent Resolution & MIME Type
When the browser encounters a `continue_on` URL, it queries `PackageManager` for an activity that can handle:
* **Action**: `Intent.ACTION_VIEW`
* **Category**: `Intent.CATEGORY_BROWSABLE`
* **Data URI**: Matching the `continue_on` URL
* **MIME Type**: `application/web-identity+json`

If a matching app is installed and verified via Digital Asset Links (`delegate_permission/common.use_as_origin`), the browser launches the activity directly.

### 2. AndroidManifest.xml Example
```xml
<activity
    android:name=".FedCmContinuationActivity"
    android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="idp.example"
              android:pathPrefix="/continue" />
        <data android:mimeType="application/web-identity+json" />
    </intent-filter>
</activity>
```

### 3. Native Equivalent of `IdentityProvider.resolve()`
When the user completes the authentication flow inside the native IdP activity, the app returns the resulting ID assertion token back to the browser via `Activity.setResult()`:

```java
// Inside FedCmContinuationActivity after successful user sign-in
Intent resultIntent = new Intent();
resultIntent.putExtra("token", idAssertionToken);
setResult(Activity.RESULT_OK, resultIntent);
finish();
```

The browser receives `Activity.RESULT_OK` and extracts the `"token"` string extra. This acts as the native equivalent of `IdentityProvider.resolve(token)`, immediately resolving the pending `navigator.credentials.get()` promise in the relying party's web page.

## Relationship with Native App Payment Handlers

FedCM Native App IDPs build upon the architectural precedence established by [Android Native Payment Handlers](https://www.stephenmcgruer.com/native-app-payment-handler/android-apps.html) (used by the Web `PaymentRequest` API). Both mechanisms solve a fundamental challenge: **allowing a Web API in the browser to discover, verify, and delegate credential or transaction workflows to an installed native Android application representing a web origin.**

### Shared Architectural Patterns
1. **Zero-Configuration Relying Parties**: Relying parties call standard Web APIs (`navigator.credentials.get()` or `new PaymentRequest()`) with HTTPS config URLs. The browser transparently detects whether an installed Android application can fulfill the request without relying party code changes.
2. **Bi-Directional Origin Verification**: Both systems require two-way cryptographic verification before invoking an app:
   * The web origin must declare the Android application's package name and certificate fingerprint in `.well-known/assetlinks.json`.
   * The Android application must declare support for the origin in its `AndroidManifest.xml`.
3. **Activity Result Protocol**: When launching interactive UX (such as a payment sheet or continuation sign-in screen), both APIs use Android Activity results (`Activity.setResult(Activity.RESULT_OK, resultIntent)`) to return cryptographic tokens or responses back to the browser tab.

### Intentional Architectural Simplifications in FedCM
While Payment Handlers established the foundation, FedCM Native App IDPs introduce several intentional simplifications:

* **Origin Verification (DAL)**: Payment Handlers require both Digital Asset Links (`handle_all_urls`) **and** an HTTP-fetched Payment Method Manifest (`payment-manifest.json`). FedCM requires only Digital Asset Links (`delegate_permission/common.use_as_origin`), eliminating extra network requests since `use_as_origin` is already widely deployed by Identity Providers for AuthTab and Custom Tabs.
* **App Discovery & Filtering**: Payment Handlers use `<meta-data>` tags (`default_payment_method_name` or `@array`) in `AndroidManifest.xml` to filter candidates before verification. FedCM queries all services declaring `org.w3.FedCM` and relies strictly on DAL verification (`delegate_permission/common.use_as_origin`) to filter authorized apps, keeping manifest declarations minimal.
* **IPC Mechanism**: Payment Handlers use formal AIDL interfaces (`IsReadyToPayService.aidl`) and typed Android Parcelables. FedCM uses Android `Messenger` and `Bundle` with JSON strings (`request` / `reply`), providing a lightweight protocol that avoids AIDL version-skew across independent IdP app updates while reusing FedCM's standardized JSON schemas.

### References and Further Reading
* [Android Payment Apps Developer Guide (`web.dev`)](https://web.dev/articles/android-payment-apps-developers-guide?hl=en)
* [Native App Payment Handler Specification (Stephen McGruer)](https://www.stephenmcgruer.com/native-app-payment-handler/spec.html)
* [Android Native App Payment Handlers Guide (Stephen McGruer)](https://www.stephenmcgruer.com/native-app-payment-handler/android-apps.html)
* [Android Payment Apps Origin Delegation (`web.dev`)](https://web.dev/articles/android-payment-apps-delegation)
* [Intent to Ship: PaymentRequest on WebView (`blink-dev`)](https://groups.google.com/u/1/a/chromium.org/g/blink-dev/c/1Ep8DvUHU1Q/m/nvB3DSk1BgAJ)
* [Intent to Ship: Android Payment Apps (`blink-dev`)](https://groups.google.com/u/0/a/chromium.org/g/blink-dev/c/GR-MdDaKCoA/m/YOcIo6mUBgAJ)

## WebViews

We currently have FedCM disabled in WebViews, and I think that, with this mechanism, we would be able to enable it.

Because FedCM, running in the WebView’s process space can leverage the native application state directly to gather the users account, and has the ability to then use the continuation API in its own address space, we would enable federation to work on WebViews with the following properties:

- The user isn’t required to login (or to be logged in) to the IdP in the calling App’s process space (e.g. in the WebView’s cookie jar)  
- The IdP shares with the calling App’s process space an JWT / access token, which is narrowly scoped to the specific app, allowing the user to login to the App with the IdP  
- The IdP shares with the calling App’s process space enough information to construct an account chooser (namely, the user’s name/email/picture)

Because FedCM is already distributed with JS SDK across without requiring websites (and apps) to change, this can be deployed at scale to all apps using webviews.

# Security Considerations

## App Origin Authentication

One of the most important security considerations is how to find and authenticate the right application, given a URL.

On the Web, every FedCM request relies on HTTP/DNS/TLS to authenticate the right server. Android Apps, on the other hand, don’t have the same naming system.

How does the browser, then, figure out which Android native app to talk to?

How do we guarantee that the Android native app is owned by the same entity as the FedCM IdP origin?

> Entra leverages things like the reply URL, or application configuration to allow for native brokering to occur.

### Background

So, while Android doesn’t have a native naming system, what they do have, however, which we argue here is sufficient, is a good and reliable bi-directional mapping between URLs and Android Package names, via [Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started) and [Android’s Verification System](https://developer.android.com/training/app-links/verify-applinks).

This is a long and intricate system, but at its core, it relies on a bidirectional statement to bind an origin to an app: (a) the https origin needs to point to the app and (b) the app needs to point to the https origin.

The way the first statement (a) is made is with [Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started): a well-known file (e.g. https://www.example.com/.well-known/assetlinks.json) that is hosted in an origin that describes what apps can handle what urls.

For example, the following .well-known file states that, for “[https://www.example.com](https://www.example.com)” the "com.example.app” android package name (with a given sha256\_cert\_fingerprints), can handle a well defined relationship, "delegate\_permission/common.handle\_all\_urls":

```json
[{
  "relation": ["delegate_permission/common.use_as_origin"],
  "target" : { "namespace": "android_app", "package_name": "com.example.app",
               "sha256_cert_fingerprints": ["hash_of_app_certificate"] }
}]
```

The second statement (b) is made in the AndroidManifest.xml file that is shipped with every Android App.

For example, the following snippet in and AndroidManifest.xml file states that “com.example.app” can handle “[https://www.example.com/products](https://www.example.com/products)\*” urls:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />               
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data
    android:scheme="https"
    android:host="www.example.com"
    android:pathPrefix="/products" />
</intent-filter>
```

If only statement (b) was made, without a corresponding (a) from the origin, any malicious app would be able to intercept and handle urls from other apps (e.g. “my.malicoius.app” could intercept “[https://facebook.com](https://facebook.com)” urls). If only statement (a) was made, without a corresponding (b) statement, any malicious origin could drive traffic to any app (e.g. “[https://malicious.com](https://malicious.com)” could make every link clicked on Android go to the wrong app).

When both (a) and (b) are used in conjunction, Android is able to verify that a specific app’s intent filter is responsible for handling a specific set of URLs.

This is used on a variety of things, but most notably on deep-linking: if an android user using an email client app clicks on [https://www.example.com/products/1234.html](https://www.example.com/products/1234.html), the user is directed to “com.example.app” rather than the default browser.

###

### Proposal

We propose to reuse this trust signal for FedCM bound services: we are going to require that the app has already set up verified links for the FedCM endpoints before we call into it.

So, for example, the FedCM IdP would need to have the following declarations: the DAL links in their origin, the \<intent-filter\> in their AndroidManifest file and the definition of the service.

When those three conditions are met, the browser is able to connect to a service in an Android App that is guaranteed to represent the origin.

In Chrome’s implementation, we’ll re-use the ChromeOriginVerifier for [AuthTab](https://source.chromium.org/chromium/chromium/src/+/main:chrome/android/java/src/org/chromium/chrome/browser/browserservices/ui/controller/AuthTabVerifier.java;drc=62a9a00176f862e688fa47919ed6f19b59f77b5f;l=122) and DomainVerificationManager for Android S and above.

# Privacy Considerations

We don’t think there are any privacy considerations that go beyond what has already been considered in FedCM: the ability to talk to Android native apps only changes the transport, but maintains the privacy properties of the protocol that’s running on top of it.

# Open Questions

1) Can background services make external HTTP requests?  
   1) Yes, just out of the main thread, so need an Executor  
  
