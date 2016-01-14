---
layout: post
title: Create Chooser Intent with Packages Excluded
comments: true
---

It's extremely easy to [share with Intent on Android](http://android-developers.blogspot.com/2012/02/share-with-intents.html). However, there are some apps that capture the `ACTION_SEND` intents, but [doesn't allow](https://developers.facebook.com/bugs/332619626816423) the app to pre-fill text set with `EXTRA_TEXT`, resulting in poor user experience.

With the following code, we can easily exclude some unwanted apps from the chooser intent:
{% highlight java %}
@Nullable
private static Intent createChooserExcludingPackage(
        Context context, String packageToExclude, String text) {
    // 1) gets all activities that can handle the sharing intent
    final Intent sendIntent = new Intent(Intent.ACTION_SEND)
            .setType("text/plain");
    final PackageManager pm = context.getPackageManager();
    final List<ResolveInfo> resolveInfoList = pm.queryIntentActivities(sendIntent, 0);
    final int size = resolveInfoList.size();
    if (size == 0) {
        return null;
    }

    // 2) now let's filter by package name
    final ArrayList<Intent> filteredIntents = new ArrayList<>(size);
    for (int i = 0; i < size; ++i) {
        final ResolveInfo resolveInfo = resolveInfoList.get(i);
        final String packageName = resolveInfo.activityInfo.packageName;
        if (!packageToExclude.equals(packageName)) {
            // creates a LabeledIntent with custom icon and text
            final LabeledIntent labeledIntent = new LabeledIntent(
                    packageName, resolveInfo.loadLabel(pm), resolveInfo.getIconResource());
            labeledIntent.setAction(Intent.ACTION_SEND).setPackage(packageName)
                    .setComponent(new ComponentName(packageName, resolveInfo.activityInfo.name))
                    .setType("text/plain").putExtra(Intent.EXTRA_TEXT, text);
            filteredIntents.add(labeledIntent);
        }
    }

    // 3) creates new chooser intent
    final Intent chooserIntent = Intent.createChooser(filteredIntents.remove(0),
            context.getText(R.string.text_share_with));
    final int extraIntents = filteredIntents.size();
    if (extraIntents > 0) {
        chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS,
                filteredIntents.toArray(new Parcelable[extraIntents]));
    }
    return chooserIntent;
}
{% endhighlight %}

The above code is taken from [here](https://github.com/ZionSoft/Obadiah/commit/1aa563d5a190f8d8740010db8bdbf40d59996086). Enjoy and happy coding!
