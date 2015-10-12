---
layout: post
title: Displaying Images Efficiently on Android
comments: true
---

In this blog post, we demonstrate some simple ways to efficiently display images.

## The Straightforward Approach

The simplest way to load an image (e.g. from resource), is to load it in an `AsyncTask`, and updates the `ImageView` in `onPostExecute()`:

{% highlight java %}
class ImageLoadingTask extends AsyncTask {
    ImageLoadingTask(Resources resources, ImageView imageView) {
        mResources = resources;
        mImageView = new WeakReference(imageView);
    }
 
    @Override
    protected Bitmap doInBackground(Integer[] params) {
        return BitmapFactory.decodeResource(mResources, params[0]);
    }
 
    @Override
    protected void onPostExecute(Bitmap result) {
        final ImageView imageView = mImageView.get();
        if (imageView != null)
            imageView.setImageBitmap(result);
    }
 
    private final Resources mResources;
    private final WeakReference mImageView;
}
{% endhighlight %}

This works fine until you try to load some large image, when the image refuses to be loaded with this error from logcat:
`W/OpenGLRenderer﹕ Bitmap too large to be uploaded into a texture (4000×3726, max=2048×2048)`

And of course, it might even throw an `OutOfMemory` error on low-end devices.

## Load a Scaled Image

To solve the problem, we should request the decoder to subsample the original image, as shown below with the updated `doInBackground()`:
{% highlight java %}
protected Bitmap doInBackground(Integer... params) {
    // extracts the size of the original image first
    final int resourceId = params[0];
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(mResources, resourceId, options);
 
    // calculates the inSampleSize
    if (options.outWidth > mTargetWidth || options.outHeight > mTargetHeight) {
        options.inSampleSize
            = Math.min(Math.round((float) options.outWidth / (float) mTargetWidth),
                Math.round((float) options.outHeight / (float) mTargetHeight));
    }
 
    // decodes the image
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(mResources, resourceId, options);
}
{% endhighlight %}

This works pretty fine in most cases, until the `ImageView` is used e.g. inside a `ListView`, which recycles child views for performance reasons. In this scenario, each time the `ImageView` is displayed, it will trigger an image load task. However, the sequence when the tasks are finished is undefined. So there is a chance that the image displayed actually comes from the previous item, which for some reason takes longer to load.

## ListView

To solve this issue, the ImageView should remember the last image it’s supposed to load, so we extends the class and introduces a simple `loadImageResource()` method as below:

{% highlight java %}
public void loadImageResource(int resId) {
    // if loading or already loaded the same resource, do nothing
    // otherwise cancel the current loading task
    if (mResId == resId)
        return;
    if (mImageLoadingTask != null) {
        final ImageLoadingTask loadingTask = mImageLoadingTask.get();
        if (loadingTask != null)
            loadingTask.cancel(true);
    }
 
    mResId = resId;
    setImageBitmap(null); // or some placeholder image
 
    final ImageLoadingTask loadingTask = new ImageLoadingTask(getResources(), this);
    mImageLoadingTask = new WeakReference(loadingTask);
    loadingTask.execute(resId);
}
{% endhighlight %}

## Cache Images

Now everything works fine, except that whenever our `ImageView` get recycled, it needs to load the image again, so we introduce a simple image cache:

{% highlight java %}
public class ImageCache extends LruCache<Integer, Bitmap> {
    public ImageCache(int maxSize) {
        super(maxSize);
    }
 
    @Override
    protected int sizeOf(Integer key, Bitmap bitmap) {
        // uses byte sizes of the bitmaps
        return bitmap.getRowBytes() * bitmap.getHeight();
    }
}
 
// sample code to create an ImageCache using 1 / 8th of the memory
final int cacheSize = (int) (Runtime.getRuntime().maxMemory() / 8L);
ImageCache imageCache = new ImageCache(cacheSize);
{% endhighlight %}

With the above simple `ImageCache`, one can easily caches images in memory, and improves the performance significantly.
