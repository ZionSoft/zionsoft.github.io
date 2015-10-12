---
layout: post
title: Play with Google Play Services 1 - Set Up
---

Google has offered a bunch of services that developers can use in their Android apps. In this and following few blog posts, we demonstrate how easy it is to use them. Now, let's get everything setup.

## Download the SDK

You can find Google Play services in the Extras section from your Android SDK Manager. Simply install it. If you also want to support Froyo (Android 2.2), also install Google Play services for Froyo.

After installation, you can find it at *<android-sdk>/extras/google/google_play_services/*.

## Configure Your Project

Update the *build.gradle* file of the corresponding module, adding the following section:

{% highlight sh %}
dependencies {
    compile "com.google.android.gms:play-services:4.1.32"
}
{% endhighlight %}

Then, update the AndroidManifest.xml file of the module, adding the following section:

{% highlight xml %}
<application>
    ...
    <meta-data
        android:name="com.google.android.gms.version"
        android:value="@integer/google_play_services_version"/>
</application>
{% endhighlight %}

## Check If the Device Has Google Play Services

Considering the variety of Android devices out in the market, it’s possible that the device doesn’t have the latest version of Google Play services you required, the user disables it, or it can be simply missing from from the device. Therefore, it’s best practice to check its availability:

{% highlight java %}
@Override
protected void onResume() {
    super.onResume();
 
    // code that doesn't require Google Play services
 
    if (!checkGooglePlayServices())
        return;
 
    // code that requires Google Play services
}
 
private boolean checkGooglePlayServices() {
    int result = GooglePlayServicesUtil.isGooglePlayServicesAvailable(this);
    if (result == ConnectionResult.SUCCESS) {
        // the device has the latest Google Play services installed
        return true;
    } else {
        Dialog errorDialog = GooglePlayServicesUtil.getErrorDialog(result, this, 8964,
            new DialogInterface.OnCancelListener() {
                @Override
                public void onCancel(DialogInterface dialog) {
                    finish();
                }
            });
        if (errorDialog != null) {
            // the problem can be fixed
            // e.g. the user needs to enable or download the latest version
            errorDialog.show();
        } else {
            // for some reason, the problem can't be fixed
            // you should provide some notice to the user
        }
        return false;
    }
}
 
@Override
protected void onActivityResult (int requestCode, int resultCode, Intent data) {
    // you don't have to check it here,
    // since the onResume() callback will be invoked anyway
    if (requestCode == 8964) {
        // this is invoked when user finishes operation
        if (resultCode == Activity.RESULT_OK) {
            // problem fixed, now you can use Google Play services
        } else {
            // problem not fixed, e.g. user decides not to install the latest version
        }
    } else {
        super.onActivityResult(requestCode, resultCode, data);
    }
}
{% endhighlight %}

**Tip**: You should make your app usable and requires it only when needed, if Google Play services is not a must-have feature.

Yep, that's it. Now you have Google Play services ready, have fun with it and keep reading ;)
