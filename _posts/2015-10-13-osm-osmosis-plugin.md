---
layout: post
title: Write Own Osmosis Plugin
comments: true
---

[Osmosis](http://wiki.openstreetmap.org/wiki/Osmosis) is a command line application for processing [Open Street Map](http://www.openstreetmap.org) data, with the souce code available [here](https://github.com/openstreetmap/osmosis). The tool provides e.g. ability to generate and read data dumps, extract data inside a bounding box, etc. You can also easily write your own plugin to convert OSM data to e.g. formats your application understands.

Now, let's start.

## Write a Simple Plugin

An *osmosis* plugin is basically a normal jar, plus a `plugin.xml` file that describes how to load the plugin as shown below:

{% highlight xml %}
<?xml version="1.0" ?><!DOCTYPE plugin PUBLIC "-//JPF//Java Plug-in Manifest 1.0"
    "http://jpf.sourceforge.net/plugin_1_0.dtd">
<!-- Here, provides a unique ID and version for your plugin. -->
<plugin id="MyPlugin" version="1.0">
    <requires>
        <import plugin-id="org.openstreetmap.osmosis.core.plugin.Core" reverse-lookup="false" />
    </requires>

    <runtime>
        <library id="mycode" path="/" type="code" />
    </runtime>

    <!-- Describes where the plugin plugs into. -->
    <extension id="MyPlugin"
        plugin-id="org.openstreetmap.osmosis.core.plugin.Core" point-id="Task">
        <parameter id="name" value="MyPlugin" />
        <!-- Here, give the fully qualified name for your class loader. -->
        <parameter id="class" value="net.zionsoft.sample.osmosisplugin.MyPluginLoader" />
    </extension>
</plugin>
{% endhighlight %}

As we can see, the plugin is loaded by `MyPluginLoader`:

{% highlight java %}
public class MyPluginLoader implements PluginLoader {
    @Override
    public Map<String, TaskManagerFactory> loadTaskFactories() {
        // the map describes how to load the plugin
        // with the following statement, you can load the task
        // created by MyFactory using:
        // osmosis --read-pbf latest.osm.pbf --my-plugin
        // you can also create multiple factories that create
        // different tasks if needed
        HashMap<String, TaskManagerFactory> map = new HashMap<>();
        map.put("my-plugin", new MyFactory());
        return map;
    }
}
{% endhighlight %}

The following shows `MyFactory` that creates the real tasks:

{% highlight java %}
public class MyFactory extends TaskManagerFactory {
    @Override
    protected TaskManager createTaskManagerImpl(TaskConfiguration taskConfiguration) {
        // the provided configuration includes the argument you pass to osmosis,
        // which can be used to config the task to be created
        // e.g. if you started osmosis like this:
        // osmosis --read-pbf latest.osm.pbf --my-plugin key=value
        // you can get fetch the passed argument like this:
        // String value = getStringArgument(taskConfiguration, "key", null);

        MyTask myTask = new MyTask();
        return new SinkManager(taskConfiguration.getId(), myTask, taskConfiguration.getPipeArgs());
    }
}
{% endhighlight %}

Now, a simple task that does nothing but print some messages:

{% highlight java %}
public class MyTask implements Sink {
    @Override
    public void initialize(Map<String, Object> map) {
        // initializes resources you need
        System.out.println("initialize()");
    }

    @Override
    public void process(EntityContainer entityContainer) {
        // processes each entity
        Entity entity = entityContainer.getEntity();
        EntityType entityType = entity.getType();
        System.out.println("process(): " + entityType);

        switch (entityType) {
            case Bound:
                Bound bound = (Bound) entity;
                System.out.println(bound.toString());
                break;
            case Node:
                Node node = (Node) entity;
                System.out.println(node.toString());
                break;
            case Way:
                Way way = (Way) entity;
                System.out.println(way.toString());
                break;
            case Relation:
                Relation relation = (Relation) entity;
                System.out.println(relation.toString());
                break;
        }
    }

    @Override
    public void complete() {
        // makes sure all info is fully persisted
        System.out.println("complete()");
    }

    @Override
    public void release() {
        // releases resources
        System.out.println("release()");
    }
}
{% endhighlight %}

## Installation

Just copy the generated jar to `~/.openstreetmap/osmosis/plugins` folder.

## Use the Plugin

You can now use your plugin: `osmosis --read-pbf latest.osm.pbf --my-plugin`, and you will get some output like below:

{% highlight sh %}
Oct 13, 2015 9:05:39 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Osmosis Version 0.42-6-gf39a160-dirty
Oct 13, 2015 9:05:40 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Preparing pipeline.
Oct 13, 2015 9:05:40 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Launching pipeline execution.
initialize()
Oct 13, 2015 9:05:40 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline executing, waiting for completion.
process(): Bound
Bound(top=60.2359955, bottom=60.2358955, left=24.7136472, right=24.7186472)
process(): Node
Node(id=506188159, #tags=0)
process(): Node
Node(id=518411565, #tags=0)
process(): Node
Node(id=2009423577, #tags=0)
process(): Way
Way(id=23123806, #tags=2)
complete()
release()
Oct 13, 2015 9:05:53 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline complete.
Oct 13, 2015 9:05:53 PM org.openstreetmap.osmosis.core.Osmosis run
INFO: Total execution time: 13088 milliseconds.
{% endhighlight %}

Yes, it's that simple to create your own *osmosis* plugin, and time to hack now!
