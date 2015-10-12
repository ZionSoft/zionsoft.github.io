---
layout: post
title: Play with Google Play Services 3 - Map
comments: true
---

If you haven’t set up Google Play services yet, please follow [this tutorial](/2014/02/google-play-services-set-up/).

## Get Maps API Key

First, create a project for your app in [Google Developers Console](https://console.developers.google.com/project/), and enable *Google Maps Android API v2*.

Then get you the signing certificate’s fingerprint:
{% highlight sh %}
# displays fingerprint of your debug certificate
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
 
# displays fingerprint of your release certificate
keytool -list -keystore <your keystore>
{% endhighlight %}

Then use the SHA-1 fingerprint and your app’s package name to generate a new key for API access from Google Developers Console.

## Update AndroidManifest.xml

Now update your app's *AndroidManifest.xml* file, specifying the permissions, feature required, and the map key:
{% highlight xml %}
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="com.google.android.providers.gsf.permission.READ_GSERVICES"/>
 
<uses-feature
        android:glEsVersion="0x00020000"
        android:required="true"/>
 
<application>
    <meta-data
            android:name="com.google.android.maps.v2.API_KEY"
            android:value="your-map-key"/>
</application>
{% endhighlight %}

## Add a Map

The easiest way to show a map is to add a `MapFragment` (or a `SupportMapFragment` for older devices) to your activity. Another approach is to add a `MapView` to your activity or fragment.

To add a map view showing satellite maps with an XML file:
{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.gms.maps.MapView
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:map="http://schemas.android.com/apk/res-auto"
        android:id="@+id/map_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        map:mapType="satellite" />
{% endhighlight %}

Then in your activity:
{% highlight java %}
public class MainActivity extends Activity {
    private MapView mMapView;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        // if you use a map fragment, it will take care of its life-cycle management
        // if you use a map view, you need to forward all the following events to it
        mMapView = (MapView) findViewById(R.id.map_view);
        mMapView.onCreate(savedInstanceState);
    }
 
    @Override
    protected void onResume() {
        super.onResume();
 
        mMapView.onResume();
    }
 
    @Override
    protected void onPause() {
        mMapView.onPause();
 
        super.onPause();
    }
 
    @Override
    protected void onDestroy() {
        mMapView.onDestroy();
 
        super.onDestroy();
    }
 
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        mMapView.onSaveInstanceState(outState);
 
        super.onSaveInstanceState(outState);
    }
 
    @Override
    public void onLowMemory() {
        mMapView.onLowMemory();
 
        super.onLowMemory();
    }
}
{% endhighlight %}

## GoogleMap Object

Once you have the map ready, you can use the `getMap()` method (or the `getMapAsync()` to get reference to the map asynchronously) of the map view or fragment to get a `GoogleMap` object. If this method returns `null`, it means the map is not ready yet (e.g. because the Google Play services is not available). It serves as the entry point for interactions with the map:

* To configure the UI, e.g. show / hide my location button, enable / disable gestures, use the `UiSettings` object fetched using `GoogleMap.getUiSettings()` method.
* To convert between coordinates on the map and the location on the screen, use the `Projection` object fetched using `GoogleMap.getProjection()` method.
* You can uses `GoogleMap.setMyLocationEnabled()` method to show or hide the blue dot representing device's current location, but to get the current location you should [follow my previous tutorial on locations](/2014/11/google-play-services-locations-2/).
* To listen to map click events, you can simply set the corresponding listener using `GoogleMap.setOnMapClickListener()` or `GoogleMap.setOnMapLongClickListener()` method.
* You can set the camera (i.e. which part of the map should be shown) using the `GoogleMap.animateCamera()` (with animation) or `GoogleMap.moveCamera()` (no animation) method. To listener to camera changes (e.g. triggered by user), you can specify the corresponding listener using `GoogleMap.setOnCameraChangeListener()`.

Also, you can set map types (normal, satellite, etc.), enable / disable indoor maps, 3D buildings, etc. using the `GoogleMap` object.

## Markers and Info Windows

Markers can be used to indicate locations or other important information on the map, with more detailed information shown in the info window.

The following snippet adds a marker with a custom icon to the map, and displays a default info window showing “Hello, Helsinki!”:

{% highlight java %}
@Override
protected void onResume() {
    ...
    mGoogleMap = mMapView.getMap();
    BitmapDescriptor bitmap = ...
    Marker addedMarker = mGoogleMap.addMarker(new MarkerOptions()
        .position(new LatLng(60.1708, 24.9375))
        .title("Hello, Helsinki")
        .icon(BitmapDescriptorFactory.fromBitmap(bitmap)));
}
{% endhighlight %}

One thing to note when using e.g. `BitmapDescriptorFactory` is, you might need to use `MapsInitializer` to explicitly initialize the map.

To remove a marker, you can use the `Marker.remove()` method, or the `GoogleMap.clear()` method to remove all added markers, shapes, etc.

If you want to animate a marker on the map (e.g. use a custom marker to indicate current location instead of the boring blue dot), all you need is simply update its position periodically using the `Marker.setPosition()` method.

When the user clicks on a marker, by default it shows an info window with some title and snippet defined when the marker was added. But you can also listen to the click event:

{% highlight java %}
mGoogleMap.setOnMarkerClickListener(new GoogleMap.OnMarkerClickListener() {
    @Override
    public boolean onMarkerClick(Marker marker) {
        ...
    }
});
{% endhighlight %}

If the `onMarkerClick()` method returns true, it means the click event has been consumed. Otherwise, if false is returned, it will execute the default behavior, i.e. moving the camera to the marker, and showing the info window.

To provide a customized info window:

{% highlight java %}
mGoogleMap.setInfoWindowAdapter(new GoogleMap.InfoWindowAdapter() {
    @Override
    public View getInfoWindow(Marker marker) {
        // this will be first called
        // the returned view will be used as the entire info window
    }
 
    @Override
    public View getInfoContents(Marker marker) {
        // this will only be called if getInfoWindow() returns null
        // the returned view will be used inside the default info window frame
    }
});
{% endhighlight %}

The info window is **rendered as an image**. It means any subsequent changes won’t be visible unless you re-open the info window, and the info window can only listen to one click event for the whole info window.

## Circles, Polygons, and Polylines

To add those shapes to the map, you basically follow the same pattern:

{% highlight java %}
Circle addedCircle = mGoogleMap.addCircle(new CircleOptions()
    .center(new LatLng(60.1708, 24.9375))
    .radius(1000));
 
// when adding a polygon, the system will automatically connect the
// last coordinate to the first one
Polygon addedPolygon = mGoogleMap.addPolygon(new PolygonOptions()
    .add(new LatLng(60, 24), new LatLng(61, 24), new LatLng(61, 25), new LatLng(60, 25));
 
// the system won't automatically connect the coordinates
// thus you can see the difference compared to the above polygon
Polyline addedPolyline = mGoogleMap.addPolyline(new PolylineOptions()
    .add(new LatLng(60, 24), new LatLng(61, 24), new LatLng(61, 25), new LatLng(60, 25));
{% endhighlight %}

## Ground and Tile Overlays

The main difference between these two overlays is: the **ground overlay** provides a single image for a fixed location on the map, while the **tile overlay** provides a set of images for different locations at different zoom level on the map.

To add a ground overlay:

{% highlight java %}
GroundOverlay addedGroundOverlay = mGoogleMap.addGroundOverlay(new GroundOverlayOptions()
    .image(bitmap)
    .anchor(0.5, 0.5)
    .position(new LatLng(60.1708, 24.9375), 100, 100));
{% endhighlight %}

To add a tile overlay:

{% highlight java %}
TileProvider tileProvider = new TileProvider() {
    @Override
    public Tile getTile (int x, int y, int zoom) {
        // a Tile object consists of raw image data, as well its width and height
        // the map is divided into a 2^zoom x 2^zoom grid
        // for each grid, the system targets 256 x 256 dp
        // if TileProvider.NO_TILE is returned,
        // it means you don't want to provide tiles yourself
        // if null is returned, it means the tile could not be found now,
        // and will be tried again later
    }
};
TileOverlay addedTileOverlay = mGoogleMap.addTileOverlay(new TileOverlayOptions()
    .tileProvider(tileProvider));
{% endhighlight %}

That’s it for today. Happy hacking and keep reading :)
