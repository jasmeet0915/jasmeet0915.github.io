---
title: "A Beginner's Guide to Making System Plugins for Gazebo - Part 1: Introductions"
weight: 1
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 1
---

{{< lead >}}
This is a 2-part series. This part covers the basics, the next part covers plugin development. If you know the basics, feel free to skip to the [next ↗](/blogs/beginner-gazebo-system-plugin-guide-2/).
{{< /lead >}}

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

## Before we Start

Before getting down to it, let's quickly address the question: "Why does it matter?"

If you are reading this blog, I am guessing you might be:
* A diligient Robotics/Simulations engineer looking to extend Gazebo for your unique use case. Maybe you want to simulate your own high fidelity sensor, or, a solar-based power system for your outdoor robot, or maybe you want Gazebo to use your own freakin' physics engine.
* A kind-hearted developer who wants to give back to the robotics community by contributing to one of the biggest open source robotics projects.
* Or, someone who just craves knowledge and like to read (You are a rare species and are very much welcome!)

I indentify myself as 2 of the above and am always trying my best to also be the third. I started working on advanced robotics back when the whole world was on shutdown with quarantine. Gazebo was my only source to research, experiment and develop projects. Naturally, this pushed me to build simulation-heavy robotics projects - either personally or professionaly - and mostly for unique use cases requiring custom additions to Gazebo.

I am also an active contributor to Gazebo starting with [this](https://github.com/gazebosim/gz-sim/issues/1057) fix in the **JointStatePublisher** System Plugin, a bunch of documentation fixes in Tutorial Parties, [this](https://github.com/gazebosim/gz-sim/issues/1909) migration of the **LensFlare** System Plugin from Classic Gazebo to the New Gazebo, and much more.

{{< alert "star" >}}
**Quick Humble Brag:** My current tutorial party collection is at a total of [3 free t-shirts](https://x.com/debounSingh/status/2005638967157563865) 💪
{{< /alert >}}

All this lead me to pursue GSoC with Open Robotics back in 2023 where I developed a feature allowing Gazebo to automatically compute Moments of Intertia for SDFormat Links (read the official press release [here](https://www.openrobotics.org/blog/2023/6/14/2023-google-summer-of-code-students)).

Alomst all of the work described above is somewhat related to System Plugins in Gazebo. I can't stress this enough when I say: **"You can do anything in Gazebo if you know how to develop plugins"**.

**Want to simulate the Martian Environment for developing your mars rover? Easy, [develop plugins]((https://www.youtube.com/watch?v=K7hV-B1lwzw&t=2566s))** that can generate terrain based on real elevation data, simulate day/light conditions and dust storms that also affect the sensors:
{{<figure src="mars_curiosity_rover.gif" width=800 loading="eager" align="center" caption="Demo gif of our team's submission to the NASA Summer Sprint Simulation Challenge, 2024. You can watch the full showcase of the project from Gazebo Community Meeting from the link above">}}

**Want more control for simulating realistic scenarios with humans around your robots? Easy, [develop a plugin](https://github.com/blackcoffeerobotics/gazebo-ros-actor-plugin)** that allows you to control actors in simulation similar to how you would control your robots:
{{<figure src="actor_vel.gif" width=800 loading="eager" align="center" caption="Demo gif of the gazebo-ros-actor-plugin. Credits: Black Coffee Robotics">}}

**Want your robots to simulate LEDs/Indicators in Gazebo to test all your software pathways? Well, you guessed it, [develop a plugin](https://github.com/jasmeet0915/gz_sim_led_plugin)** for it:
{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

Now that we have established the significance of developing plugins with Gazebo, let's quickly brush over what this blog series offer. Like the title suggests, the aim is to provide a **"Beginner's guide to making System Plugins for Gazebo"** - from introductions to development details.

Throughout this series, I'll be taking references from my latest plugin, [**gz_sim_led_plugin**](https://github.com/jasmeet0915/gz_sim_led_plugin), which allows you to simulate LEDs/Indicators in Gazebo (the one mentioned above) for explaining implementation details and idealogies. This part, however, is more about the introductions, terminologies and concepts. So feel free to skip if you are already aware of all that!

{{< alert "github" >}}
Consider trying out the [plugin]([`gz_sim_led_plugin`](https://github.com/jasmeet0915/gz_sim_led_plugin)). Issues, PRs, Stars are welcome :)
{{< /alert >}}

<br>
{{< github repo="jasmeet0915/gz_sim_led_plugin" showThumbnail=false >}}

Let's start!

## A (Very) Brief Introduction to Gazebo
In this section, we'll discuss all the precursors you need for making your own plugins in Gazebo.

### First things first, What is Gazebo?
Well, its a Robotics Simulator, duh.

But that is a user's perspective. What about from a developer's perspective? From a developer's perspective, Gazebo is a collection of [numerous C++ libraries](https://gazebosim.org/libs/) (16 to be precise) that work together to make every thing possible from physics to rendering to GUI, CLI, SDFormat parsing, etc.

Libraries such as [**gz-physics**](https://github.com/gazebosim/gz-physics), [**gz-rendering**](https://github.com/gazebosim/gz-rendering) act as an abstraction that allows Gazebo to work with different physics and rendering engines making it hugely customizable. Libraries such as [**gz-sim**](https://github.com/gazebosim/gz-sim) and [**gz-gui**](https://github.com/gazebosim/gz-gui) act as the backend and frontend respectively, while [**gz-transport**](https://github.com/gazebosim/gz-transport) and [**gz-msgs**](https://github.com/gazebosim/gz-msgs) enables the communication between different sub systems of Gazebo using topics and services (much like ROS / ROS 2). I can go on but I hope you get the idea.

### Gazebo's Architecture, Terminologies and Plugins
There is a very handy official doc from Gazebo which explains the complete [architecture](https://gazebosim.org/docs/latest/architecture/) and [terminologies](https://gazebosim.org/api/gazebo/3/terminology.html) in detail. For keeping things short here, I would suggest you read the linked docs with some extra emphasis on **Gazebo's client-server** & **plugin-based architecture**, **Entities**, **Components**, **Entity Component Manager (ECM)**, and **Simulation Loop**.

## Gazebo's System Plugins & Plugin Interfaces

A plugin, by definition, is a piece of software which can be "plugged in" to an existing software to modify its runtime behavior. Gazebo also offers a similar plugin-based architecture where we can load such **System Plugins** dynamically in association with any Entity in our simulation world. You can find a list of all the system plugins that come with Gazebo by default [here](https://github.com/gazebosim/gz-sim/tree/gz-sim10/src/systems).

### Loading & Execution

Whenever Gazebo parses an SDF world, it looks for the:

```xml
<plugin filename="libMyPlugin.so" name="MyPluginClass">
...
</plugin>
```
From this snippet, the `filename` attribute provides the name of the plugin's shared library and the `name` attribute provides the name with which the plugin's class was registered in the source code. These details help Gazebo to load the plugin as an instance at runtime and call its functions to execute the extra functionality it adds.

**But, how does Gazebo execute the functionality of the plugin? Since a plugin can be made by anyone, how does Gazebo know which functions of the plugin instance to call and when?**

This is where the **Plugin Interfaces** come in. An interface, like the name implies, acts as a fixed interface between Gazebo and the plugins. They allow Gazebo to execute the plugin's functionality without caring about how they have been implemented. These interfaces are basically abstract classes and virutal functions that the plugin can implement. After loading, Gazebo calls the different interface functions for each of the loaded plugins at different points of time in the simulation loop. Below is the list of different available interfaces:

All of this loading happens in the [SystemLoader](https://github.com/gazebosim/gz-sim/blob/gz-sim10/src/SystemLoader.cc) class of **gz-sim** with the help of [gz-plugin](https://gazebosim.org/libs/plugin/) library which is yet another library in Gazebo's aresenal. On the other hand, the execution of the different interfaces of the loaded plugins is handled by the [SystemManager](https://github.com/gazebosim/gz-sim/blob/c38c6663521db89581306c48a06c9c24f35fe1dc/src/SystemManager.cc#L289) and the [SimulationRunner](https://github.com/gazebosim/gz-sim/blob/c38c6663521db89581306c48a06c9c24f35fe1dc/src/SimulationRunner.cc#L615) classes in **gz-sim**.

Below is a list of different interfaces that are available for System Plugins to implement.

### ISystemConfigure
This interface is **executed once** when the **plugin is loaded** by Gazebo. This where we get our parsed SDF description as a `sdf::ElementConstPtr` instance which we can use to configure our plugin with any user provided settings. In addition to that, this interface also receives a pointer to the `gz::sim::EntityComponentManager` which can be used to read/write any kind of data from any kind of entity you want.

**For instance,** the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L170) system plugin uses the **ISystemConfigure** to read the user settings like `<wheel_separation>`, `<wheel_radius>`, `<max_linear_velocity>`, etc. and configure the class internally by using these values for their respective members. 

### ISystemPreUpdate
This is interface is **executed periodically** by Gazebo right **before physics run** in **every step of the simulation loop**. This is where you would want your plugin to apply any changes or modifications (apply force, velocity, controls, etc.) to any entity in the simulation. You can then see the effects of that change during that simulations step. This interface receieves a `gz::sim::UpdateInfo` instance which you you can use to access things like `simTime`. Similar to the **ISystemConfigure**, you also get the `gz::sim::EntityComponentManager` here with read/write access.

**For instance,** the [ApplyJointForce](https://github.com/gazebosim/gz-sim/blob/gz-sim10/src/systems/apply_joint_force/ApplyJointForce.cc) system plugin uses the **ISystemPreUpdate** interface to apply a force to the joint entity with the name `<joint_name>` using the [JointForceCmd](https://gazebosim.org/api/sim/10/jointforcecmdcomponent.html) component.

Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L390) plugin uses the **ISystemPreUpdate** to apply velocity commands to the required joints using the **JointVelocityCmd** component.

### ISystemPostUpdate
This interface is also **executed periodically** by Gazebo right at the **end of each simulation step**. It carries the same signature as the **ISystemPreUpdate** with one difference that it is read-only, i.e, you cannot write any changes to entities in this function. This is where you would want to see the results of any actions that occur during the current simulation step like state changes (pose, velocity, etc.), sensor readings, etc.

**For instance,** the [JointStatePublisher](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/joint_state_publisher/JointStatePublisher.cc#L139) system plugin uses the **ISystemPostUpdate** interface to read the joint states and publish them on a topic at the end of every simulation step after the physics has already acted.

Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L497) plugin uses the **ISystemPostUpdate** to read the joint position and velocities for updating and publishing the odometry.

Apart from these 3 interfaces, there are also `ISystemUpdate` and `ISystemReset` interfaces that are pretty self explanatory but you can read more about them from [here](https://gazebosim.org/api/sim/10/createsystemplugins.html).

### Some other kinds of Plugins in Gazebo's Ecosystem
#### Physics Plugins
Gazebo's plugin-based architecture provides it with the fexibility of being used with any kind of physics engine. To do that, you just need to implement the required interfaces from the **gz-physics** library. This implementation will connect your physics engine with Gazebo and is known as the **Physics Plugin**. You can read more about it [here](https://gazebosim.org/api/physics/9/physicsplugin.html). By default, Gazebo comes with a bunch Physics Plugins for common physics engines like: DART and Bullet. Hoping to see a Mujoco Physics Plugin for Gazebo in future 🤞.

#### Rendering Engine Plugins
Similar to the Physics Plugins, Gazebo also allows you to integrate any Rendering Engine of your choice with the help of **Rendering Engine Plugins** and the [gz-rendering](https://github.com/gazebosim/gz-rendering) library. You can read more about it [here](https://gazebosim.org/api/rendering/8/renderingplugin.html). By default, Gazebo comes with rendering engine plugins for OGRE, OGRE2 and Optix (a very old version of Optix). This is where we might see some changes if we want "more realistic" simulations similar to Gazebo's aternatives such as Isaac Sim, O3DE, etc.

#### GUI Plugins
These plugins are loaded on the client side and extend Gazebo's GUI. You can add custom panels, widgets, or visualization tools to help interact with your simulation. **For instance,** you might want to create a plugin that displays sensor data or controls robots directly from the GUI. Basically anything you see on Gazebo's default right side panel is a GUI Plugin. These GUI plugins are made possible with the help of [gz-gui](https://github.com/gazebosim/gz-gui/tree/main) library. You can read more about it from [here](https://gazebosim.org/api/gui/9/plugins.html).

#### Rendering Plugins
You should use them when you want to change the visual appearance of your rendering scene. Here, it advised to use the [EventManager](https://gazebosim.org/api/gazebo/6/classignition_1_1gazebo_1_1EventManager.htmlA) class to connect to the events such as **PreRender**, **Render** and **PostRender** as they are emitted directly from the global render thread and safer to use. However, do keep in mind, Gazebo has 2 scenes being rendered by default: one on client-side and other on server-side. Loading your Rendering Plugin as a System Plugin, would only affect the server's scene (affecting sensors) and loading it as a GUI plugin would only affect the UI-side scene. Luckily, we have the [SceneBroadcaster System Plugin](https://github.com/gazebosim/gz-sim/tree/gz-sim10/src/systems/scene_broadcaster) which broadcasts all the changes in the scene from the server to the GUI. You can read more about Rendering Plugin [here](https://gazebosim.org/api/sim/10/rendering_plugins.html). 

#### Visual System Plugins
As another alternative to the **SceneBroadcaster** setup explained above, you can go with **Visual System Plugins**. These are a special kind of System Plugins that are loaded from within the `<visual>` element of a model's SDF. By default, they are loaded on both the server and client sides, making them ideal for plugins that need to update visuals in both the simulation and the GUI. But don't forget to **use the rendering events (explained above) to make any rendering changes to the scene.** The [ShaderParam plugin example](https://github.com/gazebosim/gz-sim/blob/gz-sim10/examples/worlds/shader_param.sdf) demonstrates how to use visual system plugins in the new Gazebo.

{{< alert "bug" >}}
**Heads Up:** My original plan with the LED Plugin was to go with the Visual System Plugins but that led me into some weird issues with gz services. You can read more about that from [this](https://github.com/gazebosim/gz-sim/issues/3207#issuecomment-3691392921) thread on github.
{{< /alert >}}

And, that's a wrap on Part 1 🎉. I hope you learned something new. Do checkout the [next part](/blogs/beginner-gazebo-system-plugin-guide-2/)!
