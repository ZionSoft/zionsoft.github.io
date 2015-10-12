---
layout: post
title: Have fun with Android Wear
comments: true
---

To use Android Wear, you’ll need a hosting device (e.g. a phone or a tablet) with Android 4.3 or above, with BLE support. To [set up the Google Play services](/2014/02/google-play-services-set-up/) that is used for communication between phone and watch, please [follow this tutorial](/2014/02/google-play-services-set-up/).

Among others, you will need to add the wearable module to your *build.gradle* file for both wearable app project and phone app project:

{% highlight sh %}
compile 'com.google.android.gms:play-services-wearable:7.0.0'
{% endhighlight %}

Then you should include your wearable app project into your phone app project:

{% highlight sh %}
debugWearApp project(path:':MyAndroidWear', configuration: 'debug')
releaseWearApp project(path:':MyAndroidWear', configuration: 'release')
{% endhighlight %}

## Basic wearable app

Basically, you can run any Android application on your watch, though it has a small display and limited hardware support (e.g. some watch has no GPS support). You can also use the support library provided by Google for some common UI widgets:

{% highlight sh %}
dependencies {
  compile 'com.google.android.support:wearable:1.1.0'
}
{% endhighlight %}

This library also provides the handy way to load a different layout for square or round watches. In your main activity’s layout file:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<android.support.wearable.view.WatchViewStub
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:app="http://schemas.android.com/apk/res-auto"
  android:id="@+id/watch_view_stub"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  app:rectLayout="@layout/rect_activity_main"
  app:roundLayout="@layout/round_activity_main"/>
{% endhighlight %}

The `WatchViewStub` view will load the corresponding layout based on the shape of the watch. Then in your activity:

{% highlight java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
 
  setContentView(R.layout.activity_main);
  WatchViewStub watchViewStub = (WatchViewStub) findViewById(R.id. watch_view_stub);
  watchViewStub.setOnLayoutInflatedListener(new WatchViewStub.OnLayoutInflatedListener() {
    @Override
    public void onLayoutInflated(WatchViewStub stub) {
      // the layout is fully inflated
    }
  });
}
{% endhighlight %}

## Send and sync data

There are two ways to share data between your phone and watch:

* Send a specific message to a certain node using the MessageApi.
* Share data among all nodes using the DataApi. With this API, the data sent will be synchronized across all connected devices, which means the data will be pushed to a disconnected device if it gets connected later.

With both methods, the data are private to the application, so the develop doesn’t need to worry about the privacy nor security.

### Send data with MessageApi

Once you have a connected `GoogleApiClient`, you can use the following code to send a message to a connected node:

{% highlight java %}
// first finds all the connected nodes
Wearable.NodeApi.getConnectedNodes(googleApiClient).setResultCallback(
  new ResultCallback<NodeApi.GetConnectedNodesResult>() {
    @Override
    public void onResult(NodeApi.GetConnectedNodesResult getConnectedNodesResult) {
      List<Node> nodes = getConnectedNodesResult.getNodes();
      if (nodes.size() == 0) {
        // no connected nodes
        return;
      }
      // sends a message to the specified path for the first connected node
      String nodeId = nodes.get(0).getId();
      byte[] data = ...;
      Wearable.MessageApi.sendMessage(googleApiClient, nodeId, "/some/random/path", data);
    }
  });
{% endhighlight %}

### Sync data with DataApi

Once you have a connected `GoogleApiClient`, you can use the following code to sync data across all connected nodes:

{% highlight java %}
PutDataMapRequest putDataMapRequest = PutDataMapRequest.create("/some/random/path");
DataMap dataMap = putDataMapRequest.getDataMap();
// you can put different data here
dataMap.putLong("key 1", 12345L);
Wearable.DataApi.putDataItem(googleApiClient, putDataMapRequest.asPutDataRequest());
{% endhighlight %}

### Receive data with WearableListenerService

To receive messages or data updates from other nodes, you must extend the `WearableListenerService`, whose life cycle is managed by the phone or the watch:

{% highlight java %}
public class WearableListener extends WearableListenerService {
  @Override
  public void onMessageReceived(MessageEvent messageEvent) {
    // this is called when it receives one single message from
    // a connected node
  }
 
  @Override
  public void onDataChanged(DataEventBuffer dataEvents) {
    // this is called when one or more data is created, updated
    // or deleted using the DataApi
    // note:
    // 1) if the same data is updated several times, you might
    // only be notified once, with the final state of the data
    // 2) it might contain more than one data events
    // 3) the provided buffer is only valid till the end of this
    // method
    for (DataEvent dataEvent : dataEvents) {
      DataItem dataItem = dataEvent.getDataItem();
      Uri dataItemUri = dataItem.getUri();
      // the path is the one provided when sync the data with DataApi
      String path = dataItemUri.getPath();
      if (!"/some/random/path".equals(path)) {
        continue;
      }
      switch (dataEvent.getType()) {
        case DataEvent.TYPE_CHANGED:
          // the data is created or updated
          // note that if it is updated several times with the
          // same data, this method won't be called
          break;
        case DataEvent.TYPE_DELETED:
          // the data is deleted
          // note that if the data is not deleted, it will be
          // "persisted" until the application is removed
          break;
      }
    }
  }
}
{% endhighlight %}

And register it in your `AndroidManifest.xml`:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="net.zionsoft.sample">
  <application>
    <service android:name=".MyWearableListenerService">
      <intent-filter>
        <action android:name="com.google.android.gms.wearable.BIND_LISTENER"/>
      </intent-filter>
    </service>
  </application>
</manifest>
{% endhighlight %}

Alternatively, for an activity or a fragment, you can use `DataApi` or `MessageApi` to register a temporary listener.

Yep, that’s it. Quite straight forward to get your app running on the watch, happy hacking!
