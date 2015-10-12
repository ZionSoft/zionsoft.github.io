---
layout: post
title: Play with Google Play Services 6 - Push Notification
---

If you haven’t set up Google Play services SDK yet, please [follow this tutorial](/2014/02/google-play-services-set-up/). Today, we demonstrate how to use the cloud messaging / push notification.

## Enable Cloud Messaging API

First, create a project in [Google Developers Console](https://console.developers.google.com/project/) for your app, and enable the *Google Cloud Messaging for Android* API. The project number will be used as the **GCM sender ID**.

Then, create a server key in Public API access with your server's IP address (for testing purposes, you can use e.g. *0.0.0.0/0* to allow anybody to send messages with the generated API key). The generated API key will be used to authenticate your server.

**Note**: For devices running Android older than 4.0.4, it requires users to set up a Google account before using GCM.

## Update AndroidManifest.xml

Now, update your *AndroidManifest.xml* file, specifying the required permissions:

{% highlight xml %}
<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE"/>
 
<permission
    android:name="com.example.app.permission.C2D_MESSAGE"
    android:protectionLevel="signature" />
<uses-permission android:name="com.example.app.permission.C2D_MESSAGE" />
{% endhighlight %}

## Register The Client

The following snippet shows how to register a client for cloud messaging:

{% highlight java %}
// this might invoke network calls, so call me in a worker thread
private void registerCloudMessaging() {
    // repeated calls to this method will return the same token
    // a new registration is needed if the app is updated or backup & restore happens
    final String token = InstanceID.getInstance(this).getToken("my-GCM-sender-ID",
        GoogleCloudMessaging.INSTANCE_ID_SCOPE, null);
 
    // then uploads the token to your server
 
    // or, if the client wants to subscribe to certain topics, use this method
    GcmPubSub.getInstance(this).subscribe(token, "/topics/your_topic_name", null);
}
{% endhighlight %}

**Note** that Google’s GCM server doesn’t handle localization nor message scheduling, so the client might also need to upload e.g. the preferred language and timezone to the server to improve user experiences.

## Receive Messages

In your *AndroidManifest.xml*, specify a listener:

{% highlight xml %}
<application ...>
    <receiver
        android:name=".MyPushNotificationReceiver"
        android:permission="com.google.android.c2dm.permission.SEND" >
        <intent-filter>
            <action android:name="com.google.android.c2dm.intent.RECEIVE" />
            <category android:name="com.example.app" />
        </intent-filter>
    </receiver>
</application>
{% endhighlight %}

Then you can handle the message yourself:
{% highlight java %}
public class MyPushNotificationReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // a null or "gcm" message type indicates a regular message,
        // and you can get the message details as described in section 5
        // "deleted_messages" indicates some pending messages are deleted by server
        String messageType = intent.getStringExtra("message_type");
 
        // indicates the sender of the message, or the topic name
        String from = intent.getStringExtra("from");
    }
}
{% endhighlight %}

## Send a Message

To send a push message, you can simply send a POST request to Google’s GCM server. The URL of the server is: *https://gcm-http.googleapis.com/gcm/send*.

The HTTP header must contain the following two:

* Authorization:key=<your-api-key>
* Content-Type: application/json

The HTTP body is a JSON object, something like this:
{% highlight json %}
{
  "to": "client_registration_id_or_topic_name",
  "data": {
    "key_1": "value_1",
    "key_2": "value_2"
  },
  "priority": "normal"
}
{% endhighlight %}

The fields of the `data` object represent the key-value pairs of the message's payload data. With the above example, the client will receive an intent with an extra of key `key_1` and value `value_1`, and another extra of key `key_2` and value `value_2`.

The `priority` can be either `normal` (default) or `high`. The messages marked as `high` priority will be sent immediately, even when the device is in [Doze](https://developer.android.com/training/monitoring-device-state/doze-standby.html) mode. For `normal` priority messages, they will be batched for devices in Doze mode, and will be discarded if the message expires while the device is in Doze mode.

For a complete list of allowed fields in the posted JSON, check [here](https://developers.google.com/cloud-messaging/http-server-ref).

The response can contain the following status code:

* 200: The message is successfully processed.
* 400: The request JSON object is malformed.
* 401: The sender fails to authenticate itself.
* 5xx: Server error, the sender should respect the `Retry-After` and retry later.

More details of how to parse the response messages can be found [here](https://developers.google.com/cloud-messaging/http-server-ref#interpret-downstream).

That’s it for today. Happy hacking and keep reading!
