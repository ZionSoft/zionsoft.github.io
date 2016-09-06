---
layout: post
title: Wrapping Existing Code With RxJava
comments: true
---

We are using [RxJava](https://github.com/ReactiveX/RxJava) in Android a lot, with good reasons. However, we still need to use code that is not built with RxJava, so let's wrap them.

## Synchronous APIs

For very simple synchronous APIs, you can use `Observable.just()` to wrap them. e.g. you can use `Observable.just(1, 2, 3, 4, 5)` to emit an integer sequence from 1 to 5.

If the API is blocking, you can use `Observable.defer()` to wrap them:
{% highlight java %}
Observable<String> observable = Observable.defer(
  new Func0<Observable<String>>() {
    @Override
    public Observable<String> call() {
      try {
        return Observable.just(getStringBlocking());
      } catch (Exception e) {
        return Observable.error(e);
      }
    }
  });
{% endhighlight %}

And if you want RxJava to do `try...catch` for you, you can use `Observable.fromCallable()`:
{% highlight java %}
Observable<String> observable = Observable.fromCallable(
  new Callable<String>() {
    @Override
    public String call() throws Exception {
      return getStringBlocking();
    }
  });
{% endhighlight %}

## Asynchronous APIs

Usually, existing code uses callbacks to support asynchronous operations, e.g. in Android we can request location updates like this:
{% highlight java %}
locationManager.requestLocationUpdates(
  LocationManager.GPS_PROVIDER, 1000L, 10.0F,
  new LocationListener() {
    @Override
    public void onLocationChanged(Location location) {
    }

    ...
  });
{% endhighlight %}

For advanced RxJava users, who don't need to read this article, you can use `Observable.create()` to wrap it, and fullfil the [contract](http://reactivex.io/documentation/contract.html) by yourself. But for us, we can easily use `Observable.fromEmitter()` to handle the case, and let the framework to help us:
{% highlight java %}
Observable<Location> observable = Observable.fromEmitter(
  new Action1<AsyncEmitter<Location>>() {
    @Override
    public void call(final AsyncEmitter<Location> emitter) {
      final LocationListener locationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
          // emits location
          emitter.onNext(location);
        }

        ...
      };

      emitter.setCancellation(new AsyncEmitter.Cancellable() { 
        @Override 
        public void cancel() throws Exception {
          // stops location updates when unsubscribed
          locationManager.removeUpdates(locationListener);
        });

      locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER,
        1000L, 10.0F, locationListener);

      // if you also emit onError() or onComplete(),
      // the framework will make sure the Observable
      // contract is fullfilled
    }
     // let the framework to worry about backpressure
  }, AsyncEmitter.BackpressureMode.BUFFER);
{% endhighlight %}

Enjoy.
