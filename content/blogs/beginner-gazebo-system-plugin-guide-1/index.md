---
title: "A Beginner's Guide to Making System Plugins for Gazebo - Part 1: Introductions"
weight: 1
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 1
---

{{< lead >}}
This is a 2-part series. This part covers the basics, the next part covers plugin development. If you already know the basics, feel free to skip to [Part 2 ↗](/blogs/beginner-gazebo-system-plugin-guide-2/).
{{< /lead >}}

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

## Before we Start

Let's quickly address the question: "Why does it matter?"

Simply said: **"You can literally do anything in Gazebo if you know how to develop plugins"**.

**Want to simulate the Martian Environment for developing your mars rover? Easy, [develop plugins]((https://www.youtube.com/watch?v=K7hV-B1lwzw&t=2566s))** that can generate terrain based on real elevation data, simulate day/light conditions and dust storms that also affect the sensors:
{{<figure src="mars_curiosity_rover.gif" width=800 loading="eager" align="center" caption="Demo gif of our team's submission to the NASA Summer Sprint Simulation Challenge, 2024. You can watch the full showcase of the project from Gazebo Community Meeting from the link above">}}

**Want a Control Panel that lets you mock hardware interfaces of your industrial AMRs? Develop a GUI plugin** with buttons or toggles for interfaces such as GPIOs, Manual Charging, Hardware Emergency, etc. for your simulated robot:
{{<figure src="mock_hardware_gui_plugin.png" width=800 loading="eager" align="center" caption="Screenshot of the Mock Hardware GUI Plugin I developed at Peer Robotics that allowed developers to, well, mock hardware interfaces and test all software flows in simulation">}}

**Want your robots to simulate LEDs/Indicators like real ones in Gazebo?** Or, maybe you want to **simulate a Disco Light for your virtual Dance Party? Well, you guessed it, [develop a plugin](https://github.com/jasmeet0915/gz_sim_led_plugin)** for it:
{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

{{<figure src="featured.gif" width=800 loading="eager" align="center" caption="No explanation needed ;)">}}

So clearly, Gazebo's System Plugins are important. Therefore, this blog series aims to provide a **"Beginner's guide to making System Plugins for Gazebo"** - from introductions to development details with handy insight here and there.

Throughout this series, I'll be taking references from my latest plugin, [**gz_sim_led_plugin**](https://github.com/jasmeet0915/gz_sim_led_plugin), for explaining implementation details.

{{< alert "github" >}}
Consider trying out the [plugin]([`gz_sim_led_plugin`](https://github.com/jasmeet0915/gz_sim_led_plugin)). Issues, PRs, Stars are welcome :)
{{< /alert >}}

<br>
{{< github repo="jasmeet0915/gz_sim_led_plugin" showThumbnail=false >}}

Let's start!

## A (Very) Brief Introduction to Gazebo
### First things first, What is Gazebo?
Well, its a Robotics Simulator, duh.

But that is a user's perspective. What about from a developer's perspective? From a developer's perspective, Gazebo is a collection of [numerous C++ libraries](https://gazebosim.org/libs/) (16 to be precise) that work together to make every thing possible from physics to rendering to GUI, CLI, SDFormat parsing, etc.

Libraries such as [**gz-physics**](https://github.com/gazebosim/gz-physics), [**gz-rendering**](https://github.com/gazebosim/gz-rendering) act as an abstraction that allows Gazebo to work with any physics and rendering engines you want. Libraries such as [**gz-sim**](https://github.com/gazebosim/gz-sim) and [**gz-gui**](https://github.com/gazebosim/gz-gui) act as the backend and frontend respectively, while [**gz-transport**](https://github.com/gazebosim/gz-transport) and [**gz-msgs**](https://github.com/gazebosim/gz-msgs) enables the communication between different sub systems of Gazebo using topics and services (much like ROS / ROS 2). I can go on but I hope you get the idea.

### Gazebo's Architecture, Terminologies and Plugins
Gazebo already has detailed official docs on its [architecture](https://gazebosim.org/docs/latest/architecture/) and [terminologies](https://gazebosim.org/api/gazebo/3/terminology.html). For keeping things short here, I'd suggest you read the linked docs with some extra emphasis on **Gazebo's client-server** & **plugin-based architecture**, **Entities**, **Components**, **Entity Component Manager (ECM)**, and **Simulation Loop**.

## Gazebo's System Plugins & Plugin Interfaces

A plugin, by definition, is a piece of software which can be "plugged in" to an existing software for modifying its runtime behavior.

Gazebo offers a plugin-based architecture where we can load such **System Plugins** dynamically in association with any **Entity** in our simulation world. You can find a list of all the system plugins that come with Gazebo by default [here](https://github.com/gazebosim/gz-sim/tree/gz-sim10/src/systems).

### Loading & Execution

Whenever Gazebo parses an SDF world, it looks for the:

```xml
<plugin filename="libMyPlugin.so" name="MyPluginClass">
...
</plugin>
```
From this snippet, the `filename` attribute provides the name of the plugin's shared library and the `name` attribute provides the name with which the plugin's class was registered in the source code. These details help Gazebo to load the plugin as an instance at runtime and call its functions to execute the extra functionality it adds.

**But...since a plugin is basically an external piece of software, how does Gazebo know which functions of the plugin instance to call and when?**

This is where the **Plugin Interfaces** come in. A plugin interface, like the name implies, acts as a fixed bridge between Gazebo and the plugins. They allow Gazebo to execute the plugin's functionality without caring about how they have been implemented. These interfaces are basically abstract classes and virtual functions that the plugin can implement. After loading, Gazebo calls these different interface functions for each of the loaded plugins at different points of time in the simulation loop.

All of this loading happens in the [SystemLoader](https://github.com/gazebosim/gz-sim/blob/gz-sim10/src/SystemLoader.cc) class of **gz-sim** with the help of [gz-plugin](https://gazebosim.org/libs/plugin/) library (yet another library in Gazebo's aresenal). On the other hand, the execution of the different interfaces of the loaded plugins is handled by the [SystemManager](https://github.com/gazebosim/gz-sim/blob/c38c6663521db89581306c48a06c9c24f35fe1dc/src/SystemManager.cc#L289) and the [SimulationRunner](https://github.com/gazebosim/gz-sim/blob/c38c6663521db89581306c48a06c9c24f35fe1dc/src/SimulationRunner.cc#L615) classes in **gz-sim**.

Below is a list of the different interfaces that are available for System Plugins to implement:

### ISystemConfigure
This interface is **executed once** when the **plugin is loaded** by Gazebo. This is where we get our parsed SDF description as a `sdf::ElementConstPtr` instance which we can use to configure our plugin with any user provided settings. In addition to that, this interface also receives a pointer to the `gz::sim::EntityComponentManager` which can be used to read/write any kind of data from any kind of entity you want.

**For instance,** the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L170) system plugin uses the **ISystemConfigure** to read the user settings like `<wheel_separation>`, `<wheel_radius>`, `<max_linear_velocity>`, etc. and configure the class internally by using these values for their respective members. 

### ISystemPreUpdate
This interface is **executed periodically** right **before physics runs** in **every step of the simulation loop**. Use this to apply any changes or modifications (apply force, velocity, controls, etc.) to any entities in the world. You can then see the effects of that change during the simulation step. This interface receieves a `gz::sim::UpdateInfo` instance which you you can use to access information like `simTime`. Similar to the **ISystemConfigure**, you also get the `gz::sim::EntityComponentManager` here with read/write access.

**For instance,** the [ApplyJointForce](https://github.com/gazebosim/gz-sim/blob/gz-sim10/src/systems/apply_joint_force/ApplyJointForce.cc) system plugin uses the **ISystemPreUpdate** interface to apply a force to the joint entity with the name `<joint_name>` using the [JointForceCmd](https://gazebosim.org/api/sim/10/jointforcecmdcomponent.html) component.

Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L390) plugin uses the **ISystemPreUpdate** to apply velocity commands to the required joints using the **JointVelocityCmd** component.

### ISystemPostUpdate
This interface is also **executed periodically** right at the **end of each simulation step**. It carries the same signature as **ISystemPreUpdate** with one difference - it is read-only, i.e, you cannot write any changes to entities in this function. Use this to see the results of any actions that occured during the simulation step like state changes (pose, velocity, etc.), sensor readings, etc.

**For instance,** the [JointStatePublisher](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/joint_state_publisher/JointStatePublisher.cc#L139) system plugin uses the **ISystemPostUpdate** interface to read the joint states and publish them on a topic at the end of every simulation step after the physics has already acted.

Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L497) plugin uses the **ISystemPostUpdate** to read the joint position and velocities for updating and publishing the odometry.

Apart from these 3 interfaces, there are also `ISystemUpdate` and `ISystemReset` interfaces that are pretty self explanatory. You can read more about them [here](https://gazebosim.org/api/sim/10/createsystemplugins.html).

### Some other kinds of Plugins in Gazebo's Ecosystem
#### Physics Plugins
Gazebo's plugin-based architecture provides it with the fexibility of being used with any physics engine. For that, you just need to implement the required interfaces from the **gz-physics** library. This implementation will connect your physics engine with Gazebo and is known as the **Physics Plugin**. You can read more about it [here](https://gazebosim.org/api/physics/9/physicsplugin.html). By default, Gazebo comes with a bunch **Physics Plugins** for common physics engines like: DART and Bullet.

#### Rendering Engine Plugins
Similar to the Physics Plugin, Gazebo also allows you to integrate any Rendering Engine of your choice with the help of **Rendering Engine Plugins** and the [gz-rendering](https://github.com/gazebosim/gz-rendering) library. You can read more about it [here](https://gazebosim.org/api/rendering/8/renderingplugin.html). By default, Gazebo comes with rendering engine plu gins for OGRE, OGRE2 and Optix (a very old version of Optix).

#### GUI Plugins
These plugins are loaded on the client side and extend Gazebo's GUI. You can add custom panels, widgets, or visualization tools to help interact with your simulation. Anything you see on Gazebo's default right side panel is a GUI Plugin. These GUI plugins are made with the help of the [gz-gui](https://github.com/gazebosim/gz-gui/tree/main) library which builds on top of [Qt](https://www.qt.io/). You can read more about it from [here](https://gazebosim.org/api/gui/9/plugins.html).

#### Rendering Plugins
You should use these when you want to change the visual appearance of your rendering scene. Here, it advised to use an [EventManager](https://gazebosim.org/api/gazebo/6/classignition_1_1gazebo_1_1EventManager.htmlA) instance to connect to the **PreRender**, **Render** or **PostRender** events as they are emitted directly from the global render thread and aare safer to use. However, do keep in mind, Gazebo has 2 scenes being rendered by default: one on client-side and other on server-side. Loading your Rendering Plugin as a System Plugin, would only affect the server's scene (affecting sensors) and loading it as a GUI plugin would only affect the UI-side scene. Luckily, we have the [SceneBroadcaster System Plugin](https://github.com/gazebosim/gz-sim/tree/gz-sim10/src/systems/scene_broadcaster) which broadcasts all the server scene changes to the GUI scene. You can read more about **Rendering Plugins** [here](https://gazebosim.org/api/sim/10/rendering_plugins.html). 

#### Visual System Plugins
As another alternative to the **SceneBroadcaster** setup explained above, you can also go with the **Visual System Plugins**. These are special kinds of System Plugins that are loaded from within the `<visual>` element of a model's SDF. By default, they are loaded on both the server and client sides, making them ideal for plugins that need to update visuals of entities. But don't forget to **use the rendering events (explained above) to make any rendering changes to the scene.** The [ShaderParam plugin](https://github.com/gazebosim/gz-sim/blob/gz-sim10/examples/worlds/shader_param.sdf) is an example of such **Visual System Plugins**.

{{< alert "bug" >}}
**Heads Up:** My original plan with the LED Plugin was to make it a **Visual System Plugin** but that led me into some interesting issues with gz services. You can read more about that in [this](https://github.com/gazebosim/gz-sim/issues/3207#issuecomment-3691392921) issue.
{{< /alert >}}

And, that's a wrap on Part 1 🎉. I hope you learned something new. Do checkout the [next part](/blogs/beginner-gazebo-system-plugin-guide-2/)!
