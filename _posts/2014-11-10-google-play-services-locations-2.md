---
layout: post
title: Play with Google Play Services 2 - Locations
comments: true
---

If you haven’t set up Google Play services yet, please follow [this tutorial](/2014/02/google-play-services-set-up/).

## Permissions

First, update your *AndroidManifest.xml* file to request the permissions for accessing locations:

{% highlight xml %}
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
{% endhighlight %}

The two permissions allow you to control the accuracy of the requested locations, and you don’t have to request both for your app. If you only request the coarse location permission, the fetched location will be obfuscated. However, if you want to use the geofencing feature, you must request the `ACCESS_FINE_LOCATION` permission.

## Connect Location Client

With the new GoogleApiClient class, you can connect all needed services at once, and Google Play services will handle all the permission requests, etc.:

{% highlight java %}
public class MainActivity extends Activity 
        implements GoogleApiClient.ConnectionCallbacks, GoogleApiClient.OnConnectionFailedListener {
    private GoogleApiClient mGoogleClient;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
 
        // you can also add more APIs and scopes here
        mGoogleClient = new GoogleApiClient.Builder(this, this, this)
                .addApi(LocationServices.API)
                .build();
    }
 
    @Override
    protected void onStart() {
        super.onStart();
 
        mGoogleClient.connect();
    }
 
    @Override
    protected void onStop() {
        mGoogleClient.disconnect();
 
        super.onStop();
    }
 
    @Override
    public void onConnected(Bundle connectionHint) {
        // this callback will be invoked when all specified services are connected
    }
 
    @Override
    public void onConnectionSuspended(int cause) {
        // this callback will be invoked when the client is disconnected
        // it might happen e.g. when Google Play service crashes
        // when this happens, all requests are canceled,
        // and you must wait for it to be connected again
    }
 
    @Override
    public void onConnectionFailed(ConnectionResult connectionResult) {
        // this callback will be invoked when the connection attempt fails
 
        if (connectionResult.hasResolution()) {
            // Google Play services can fix the issue
            // e.g. the user needs to enable it, updates to latest version
            // or the user needs to grant permissions to it
            try {
                connectionResult.startResolutionForResult(this, 0);
            } catch (IntentSender.SendIntentException e) {
                // it happens if the resolution intent has been canceled,
                // or is no longer able to execute the request
            }
        } else {
            // Google Play services has no idea how to fix the issue
        }
    }
}
{% endhighlight %}

## Access Current Location

Once connected, you can easily fetch the current location:

{% highlight java %}
// fetch the current location
Location location = LocationServices.FusedLocationApi.getLastLocation(mGoogleClient);
{% endhighlight %}

**Note** that this `getLastLocation()` method might return null in case location is not available, though this happens very rarely. Also, it might return a location that is a bit old, so the client should check it manually.

## Listen to Location Updates

Let’s extend the above code snippet:

{% highlight java %}
public class MainActivity extends Activity
        implements GoogleApiClient.ConnectionCallbacks, GoogleApiClient.OnConnectionFailedListener,
        LocationListener {
    ...
 
    @Override
    public void onConnected(Bundle dataBundle) {
        ...
 
        // start listening to location updates
        // this is suitable for foreground listening,
        // with the onLocationChanged() invoked for location updates
        LocationRequest locationRequest = LocationRequest.create()
                .setPriority(LocationRequest.PRIORITY_BALANCED_POWER_ACCURACY)
                .setFastestInterval(5000L)
                .setInterval(10000L)
                .setSmallestDisplacement(75.0F);
        LocationServices.FusedLocationApi.requestLocationUpdates(mGoogleClient, locationRequest, this);
    }
 
    @Override
    public void onLocationChanged(Location location) {
        // this callback is invoked when location updates
    }
 
    ...
}
{% endhighlight %}

The difference between `setFastestInterval()` and `setInterval()` is:

* If the location updates is retrieved by other apps (or other location request in your app), your `onLocationChanged()` callback here won’t be called more frequently than the time set by `setFastestInterval()`.
* On the other hand, the location client will actively try to get location updates at the interval set by `setInterval()`, which has a direct impact on the power consumption of your app.

Then how about background location tracking? Do I need to implement a long-running `Service` myself? The answer is simply, no.

{% highlight java %}
@Override
public void onConnected(Bundle dataBundle) {
    ...
    LocationRequest locationRequest = LocationRequest.create()
            .setPriority(LocationRequest.PRIORITY_BALANCED_POWER_ACCURACY)
            .setFastestInterval(5000L)
            .setInterval(10000L)
            .setSmallestDisplacement(75.0F);
    PendingIntent pendingIntent = PendingIntent.getService(this, 0,
            new Intent(this, MyLocationHandler.class),
            PendingIntent.FLAG_UPDATE_CURRENT);
    LocationServices.FusedLocationApi.requestLocationUpdates(
            mGoogleClient, locationRequest, pendingIntent);
}
{% endhighlight %}

With this, your listener (it can be an `IntentService`, or a `BroadcastReceiver`) as defined in the `PendingIntent` will be triggered even if your app is killed by the system. The location updated will be sent with key `FusedLocationProviderApi.KEY_LOCATION_CHANGED` and a `Location` object as the value in the `Intent`:

{% highlight java %}
public class MyLocationHandler extends IntentService {
    public MyLocationHandler() {
        super("net.zionsoft.example.MyLocationHandler");
    }
 
    @Override
    protected void onHandleIntent(Intent intent) {
        final Location location = intent.getParcelableExtra(FusedLocationProviderApi.KEY_LOCATION_CHANGED);
        // happy playing with your location
    }
}
{% endhighlight %}

## Geofencing

With geofencing, your app can be notified when the device enters, stays in, or exits a defined area. Please note that geofencing requires `ACCESS_FINE_LOCATION`. Again, let’s keep extending the above sample:

{% highlight java %}
public class MainActivity extends Activity
        implements GoogleApiClient.ConnectionCallbacks, GoogleApiClient.OnConnectionFailedListener {
    ...
 
    @Override
    public void onConnected(Bundle dataBundle) {
        ...
 
        // adds geofencing
        ArrayList<Geofence> geofences = new ArrayList<Geofence>();
        geofences.add(new Geofence.Builder()
                .setRequestId("unique-geofence-id")
                .setCircularRegion(60.1708, 24.9375, 1000)
                .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER
                        | Geofence.GEOFENCE_TRANSITION_DWELL
                        | Geofence.GEOFENCE_TRANSITION_EXIT)
                .setLoiteringDelay(30000)
                .build());
        PendingIntent pendingIntent = PendingIntent.getService(this, 0,
                new Intent(this, MyGeofenceHandler.class),
                PendingIntent.FLAG_UPDATE_CURRENT);
        LocationServices.GeofencingApi.addGeofences(
                mGoogleClient, geofences, pendingIntent);
    }
 
    ...
}
{% endhighlight %}

Here, the loitering delay means the `GEOFENCE_TRANSITION_DWELL` will be notified 30 seconds after the device enters the area. There’re also limitations on the number of geofences (100 per app) and pending intents (5 per app) enforced.

When a geofence transition is triggered, you can find more details easily e.g. in an `IntentService`:

{% highlight java %}
@Override
protected void onHandleIntent(Intent intent) {
    GeofencingEvent geofencingEvent = GeofencingEvent.fromIntent(intent);
    ...
}
{% endhighlight %}

Finally, to remove geofences, you can simply use one of the overloaded `LocationServices.GeofencingApi.removeGeofences()` methods.

## Mock Locations

To enable mock locations, you must first request the corresponding permission (usually for your debug build only):

{% highlight xml %}
<uses-permission android:name="android.permission.ACCESS_MOCK_LOCATION"/>
{% endhighlight %}

Then you can enable and set mock location:

{% highlight java %}
@Override
public void onConnected(Bundle dataBundle) {
    ...
    LocationServices.FusedLocationApi.setMockMode(mGoogleClient, true);
    LocationServices.FusedLocationApi.setMockLocation(mGoogleClient, mockLocation);
}
{% endhighlight %}

To distinguish if the location is a mock one, a key of `FusedLocationProviderApi.KEY_MOCK_LOCATION` in the location object’s bundle extra will be set to true.

Once done, please remember to set the mock mode to false. If you forget that, the system will set it to false when your location client is disconnected.

That’s it for today. Happy coding and keep reading.
