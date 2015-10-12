---
layout: post
title: BLE on Android
comments: true
---

Android supported [Bluetooth Low Energy / Bluetooth Smart](http://www.bluetooth.com/Pages/Bluetooth-Smart.aspx) since [4.3 / API level 18](https://developer.android.com/about/versions/android-4.3.html#Wireless). However, it’s not nice, in many ways.

## Device Scanning

To scan e.g. heart rate monitors, you’re supposed to use code like this:

{% highlight java %}
// scan for devices providing a certain service
bluetoothAdapter.startLeScan(new UUID[]{serviceUuid},
  new BluetoothAdapter.LeScanCallback() {
    @Override
    public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
      // wow, it should work
    }
  });
 
// or scan for all nearby BLE devices
bluetoothAdapter.startLeScan(new BluetoothAdapter.LeScanCallback() {
  @Override
  public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
    // yeah, it must work, right?
  }
});
{% endhighlight %}

Here, the **service UUID** is to describe the service the peripheral devices provide, e.g. *0000180d-0000-1000-8000-00805f9b34fb* is the service UUID for heart rate monitors.

Well, the reality is, the above two approaches **may or may not work**, depending on the specific device and OS version (e.g. my Samsung Galaxy S3 with Android 4.3 doesn’t support the first filter-based approach). It’s a [known issue](https://code.google.com/p/android/issues/detail?id=59490), and only fixed in Android 5.0 (only checked with Nexus 5, and seems working fine).

**So, in your code, you have to scan devices with both of the ways, and hope that one shall work.** With the second approach, you have to check if the scanned device is your target device through device names or even connect to it and figure out the services it provides.

**Sometimes, you might need to involve the users.** First, ask your users to turn off WiFi and try again. Not working? Turn off Bluetooth and on again and give a try. Still not working? OK, the Bluetooth service is crashed and can’t be restarted unless you reboot the device. So, just reboot your device like you often do with your Windows PC, then you should<sup>TM</sup> be fine.

And of course, Android and iOS hate each other, so [they refuse to talk to each other](https://code.google.com/p/android/issues/detail?id=58725).

## Paired BLE Device

If you’ve paired with BLE devices, there’s a [known bug](https://code.google.com/p/android/issues/detail?id=59262) that Android might forget the status, and you have to pair again.

## BLE and Classic Bluetooth

So you want to connect to both BLE devices and classic Bluetooth devices at the same time? Obviously, you’re asking too much. My experience is, the classic Bluetooth devices are more likely to be disconnected, and they won’t be able to connect until you ask the users to act as described above.

I haven’t figured any robust ways to get them always working. Please let me know if you’ve found solutions.

## Too Many BLE Devices

If the user has scanned too many devices, he / she will be notified *Unfortunately, Bluetooth share has stopped*. And of course it’s a [system issue](https://code.google.com/p/android/issues/detail?id=67272), and it sucks especially when there’re BLE beacons changing address all the time.

OK, there’s [a non-perfect solution](https://github.com/RadiusNetworks/bluetooth-crash-resolver). Also, for non-rooted devices, the user can turn off the Bluetooth to postpone the issue from happening. Alternatively, the user can do a factory reset when shit hits the fan, and wait for it to happen the next time. For rooted devices, you can manually edit the *bt_config.xml* file.

Best solution? Get a device with Android 4.4.3 or later.

## Concurrent Connections

There is some limitation on the number of BLE devices Android can connect to at the same time, and it seems the number is different on different versions. So better connect one by one.

## Threading Issue

This is not an Android issue, but more like a Samsung issue (works fine on Nexus, but haven't checked other vendors):
{% highlight java %}
private final BluetoothAdapter.LeScanCallback leScanCallback = new BluetoothAdapter.LeScanCallback() {
    @Override
    public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
        // on certain devices, this won't work
        // you must connect in another thread
        BluetoothGatt gatt = device.connectGatt(context, false, gattCallback);
    }
}
{% endhighlight %}

## Disconnect from GATT Server

So, hopefully, you have successfully connected to your GATT server, and after all the reading and writing, you need to disconnect:

{% highlight java %}
gatt.disconnect();
gatt.close();
{% endhighlight %}

Well, it may or may not work, and might throw exceptions, so better catch all `Throwables`.

## Conclusion

Well, it’s a big mess in 4.3 and 4.4 (thanks to the new Bluedroid Bluetooth stack they introduced in 4.2), but luckily things seem to be more stable in 5.0. And hopefully, most devices could enjoy Lollipop somewhere in the future.
