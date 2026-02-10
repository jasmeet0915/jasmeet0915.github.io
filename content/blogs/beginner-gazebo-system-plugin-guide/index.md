---
title: "A Deep Dive into How I Make System Plugins for Gazebo"
weight: 1
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 1
---

{{< lead >}}
This is going to be a long one folks! But I bet that at the end you'll have much more insights into Gazebo, and how to add your own functionality to it via custom plugins. You might also learn a thing or two about programming along the way. I'll do my best to make it worth your time :)
{{< /lead >}}

Ever since I did GSoC with Gazebo, Open Robotics back in 2023 (read the official press release [here](https://www.openrobotics.org/blog/2023/6/14/2023-google-summer-of-code-students)), a majority portion of my DMs are full of students seeking guidance on how to contribute to Gazebo and/or participate in GSoC with Open Robotics. I have also met countless number of people interested in expanding Gazebo's functionality for their specific use cases whether its work, personal projects or just for the fun of it.

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

Being someone who started working on advanced robotics when the whole world was shutdown with quarantine, Gazebo was my only source to research, experiment and develop projects. Even some of my initial work experiences (internships) were completely remote and surrounded around developing robotic applications purely on simulation. Therefore, I did my best (and am still doing) to get the hang of simulation-based robotics and how Gazebo fits into that paradigm as one most widely used open source robotics simulator.

Based on my experience, the best way to achieve this, is to get into the nitty-gritties of how it all works. Thankfully, Gazebo's open source community and modular architecture makes it a bit easier (even though the documentation can be improved) if you know where to look. Following is the pathway I took for my journey:

- Look for the issues labelled "good first issue" in the [gz-sim repo](https://github.com/gazebosim/gz-sim/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22good%20first%20issue%22) and make an attempt at fixing them.
- Participate in Gazebo Tutorial Party which happens after every release. And you get free a Gazebo t-shirt if you are on the leaderboard towards the end of of the Party. Not to brag, but I have 2 of them :)
- Then move on to the bigger issues related to open bugs, enhancements, etc.

For me, [this](https://github.com/gazebosim/gz-sim/issues/1057) was the first ever issue I fixed in gazebo which allowed a user to have custom topics for the `JointStatePublisher` system plugin. This was followed by a couple of minor fixes, Gazebo Garden's Tutorial Party and [this](https://github.com/gazebosim/gz-sim/issues/1909) issue for porting the `LensFlare` plugin from Classic Gazebo to the new Gazebo. This became my segue into the journey of developing custom plugins and features for Gazebo.

To make the segue a bit easier for others, I aim to provide a detailed guide to writing system plugins for Gazebo with this blog. I'll be taking my newest plugin, [**gz_sim_led_plugin**](https://github.com/jasmeet0915/gz_sim_led_plugin) which allows you to simulate LEDs/Indicators in Gazebo, as a reference for the same.

{{< alert "github" >}}
Do consider trying out the [plugin]([`gz_sim_led_plugin`](https://github.com/jasmeet0915/gz_sim_led_plugin)). Issues, PRs, Stars are always welcome :)
{{< /alert >}}

## A (Very) Brief Introduction to Gazebo
In this section, we are going to discuss the basic architecture and terminologies that you need as a precursor for making your own plugins. If you are already aware of the basics, feel free to skip this section.

### First things first, What is Gazebo?
Well, its a Robotics Simulator, duh. But that is a user's perspective. What about from a developer's perspective? From a developer's perspective, Gazebo is a collection of [numerous C++ libraries](https://gazebosim.org/libs/) (16 to be precise) that work together to make every thing possible from physics, rendering, GUI, CLI, SDFormat parsing, etc. Libraries such as [`gz-physics`](https://github.com/gazebosim/gz-physics), [`gz-rendering`](https://github.com/gazebosim/gz-rendering) act as an abstraction that allows Gazebo to work with different physics and rendering engine making it hugely customizable. Libraries such as [`gz-sim`](https://github.com/gazebosim/gz-sim) and [`gz-gui`](https://github.com/gazebosim/gz-gui) act as the backend and frontend respectively, while [`gz-transport`](https://github.com/gazebosim/gz-transport) and [`gz-msgs`](https://github.com/gazebosim/gz-msgs) enables the communication between different sub systems of Gazebo using topics and services (much like ROS / ROS 2). I can go on but I hope you get the idea.

### Gazebo's Architecture, Terminologies and Plugins
There is a very handy official doc from Gazebo which explains the complete [architecture](https://gazebosim.org/docs/latest/architecture/) and [terminologies](https://gazebosim.org/api/gazebo/3/terminology.html) in detail. For the sake of keeping things short here, I would suggest you read the linked docs with extra focus on understanding the following w.r.t Gazebo: client-server & plugin-based architecture, Entity, Components, Entity Component Manager (ECM), Simulation Loop, Simulation Events, [System Plugin](https://gazebosim.org/api/sim/8/createsystemplugins.html) and Plugin Interfaces(PreUpdate, PostUpdate, Configure).

### System Plugins & Interfaces:
Since this blog is about making system plugins, let's discuss about them in a bit detail.

A plugin, by definition, is a piece of software which can be "plugged in" to an existing software to modify its runtime behavior or extend its functionality. Gazebo also offers a similar plugin-based architecture where we can load such  "System Plugins" dynamically in association with any entity in our simulation world. You can find a list of all the system plugins that come with Gazebo by default [here](https://github.com/gazebosim/gz-sim/tree/gz-sim10/src/systems). Whenever Gazebo parses an SDF world, it looks for the:

```xml
<plugin filename="libMyPlugin.so" name="MyPluginClass">
...
</plugin>
```
From this snippet, the `filename` attribute provides the name of the shared library to load for the plugin and the `name` attribute provides the name with which the plugin's class was registered in the source code. These details help Gazebo to load the plugin as an instance at runtime and call its functions to reap the fruits of the extra functionality it adds. All this registering and dynamic loading of the plugins is handled with the [`gz-plugin`](https://gazebosim.org/libs/plugin/) library which is yet another library in Gazebo's aresenal.

Now the question comes, how does Gazebo execute the functionality of the plugin? Since a plugin can be made by anyone, how does Gazebo know which functions of the plugin instance to call and when?

This is where the **interfaces** come in. An interface, like the name implies, acts as a fixed interface between Gazebo and the plugins such that Gazebo can execute their functionality without caring about how they have been implemented. As long as the interfaces are there, the plugin can show its abilities. These interfaces are basically functions with a fixed signature that the plugin can implement as these are the functions which Gazebo would be calling for each of the loaded plugins at different points of time during the simulation loop. Below is the list of different available interfaces:

#### ISystemConfigure
This interface is **executed once** when the **plugin is loaded** by Gazebo. This where we get our parsed SDF description as `sdf::ElementConstPtr` which we can use to configure our plugin with any user provided settings. In addition to that, this interface also receives a pointer to the `gz::sim::EntityComponentManager` which can be used to read/write any kind of data for any kind of entity you want.

**For instance,** the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L170) system plugin uses the **ISystemConfigure** to read the user settings like `<wheel_separation>`, `<wheel_radius>`, `<max_linear_velocity>`, etc.

#### ISystemPreUpdate
This is interface is **executed periodically** by Gazebo right **before physics run** in **every step of the simulation loop**. This is where you would want your plugin to apply any changes or modifications (apply force, velocity, controls, etc.) to any entity in the simulation. You can then see the effects of that change during that simulations step. This interface receieves a `gz::sim::UpdateInfo` instance which you you can use to access things like `simTime`. Similar to the **ISystemConfigure**, you also get the `gz::sim::EntityComponentManager` here for reading/writing any information cany kind of data to any entity you want.

**For instance,** the [ApplyJointForce](https://github.com/gazebosim/gz-sim/blob/gz-sim10/src/systems/apply_joint_force/ApplyJointForce.cc) system plugin uses the **ISystemPreUpdate** interface to apply a force to the joint entity with the name `<joint_name>` using the [JointForceCmd](https://gazebosim.org/api/sim/10/jointforcecmdcomponent.html) component. Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L390) plugin uses the **ISystemPreUpdate** to apply velocity commands to the required joints using the **JointVelocityCmd** component.

#### ISystemPostUpdate
Like the name suggests, this interface is **executed periodically** by Gazebo right at the **end of each simulation step**. This interface carries the same signature as the **ISystemPreUpdate** with one difference that it is read-only, i.e, you cannot write any changes to entities in this function. Ideally this where you would want to see the results of actions that occur during the current simulation step like state changes (pose, velocity, etc.), sensor readings, etc.

**For instance,** the [JointStatePublisher](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/joint_state_publisher/JointStatePublisher.cc#L139) system plugin uses the **ISystemPostUpdate** interface to read the joint states and publish them on a topic at the end of every simulation step after the physics has already acted. Similarly, the [**DiffDrive**](https://github.com/gazebosim/gz-sim/blob/3c6627421498f66ef755a2becb2b1c3924621955/src/systems/diff_drive/DiffDrive.cc#L497) plugin uses the **ISystemPostUpdate** to read the joint position and speeds for updating and publishing the odometry.

Apart from these 3 interfaces, there are also `ISystemUpdate` and `ISystemReset` interfaces that are pretty self explanatory but you can read more about them [here](https://gazebosim.org/api/sim/10/createsystemplugins.html). In our case, we would only be using the the above 3 listed plugins.

### The special case of Visual Plugins
Now system plugins in Gazebo generally tend to avoid making changes in the visual appearances of any entities in the simulation. Why? Well, Gazebo can have upto 2 scenes at a time: one on the server side () and the other on the client (or GUI) side. Generic system plugins generally  


Now let's begin!

## Making System Plugin for Gazebo

### The LED Plugin: Understanding Our Reference

As mentioned before, I'll be taking my latest plugin, `gz_sim_led_plugin`, as a reference for all the explanations throughout this guide. As far as plugins go, this is more towards the beginner-friendly side and I believe having a working example that allows you to see everything in action, helps in better understanding.

{{< github repo="jasmeet0915/gz_sim_led_plugin" showThumbnail=false >}}

The idea behind this plugin is simple: we want to be able to simulate LEDs/Indicators in Gazebo. Why? Because all industrial systems (including robots) always have some kind of LEDs that act as indicators for whats happening behind the scenes. For instance, a robot might have LEDs that blink red when it is in fault / emergency mode which is basically the robot's way of saying "I don't feel so good :(". As a systems engineer for robots, it is your job to make sure that you map such different "modes" of the robot to different patterns / configuration of these indicators as they tell the surrounding users if, for instance, the robot: is idle and waiting for a mission, is low on battery and needs a charge, is going to reverse now so beware and keep your distance, and so on. Like any piece of hardware, simulating LEDs provides the benefit of rapid prototyping, development and testing without the need of actual hardware.

{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

### User Perspective and How Would it Work?

There are 2 things that I like to define at the beginning of writing any system plugin (or in fact any piece of software):

1) What does a user want and how would they use it?
2) A high level idea of how would it work.

#### User Perspective
For the first part, I generally try to think from a user's perspective and come up with a potential example SDF snippet for using the plugin. As we have already established, different LED patterns/configurations are generally mapped to different modes/behaviours. Therefore, a potential user of our plugin would want to be able to define a bunch of named "modes" each of which describes their own behaviour for a bunch of LEDs. So this what we have as of now:

```xml
<plugin name="plugin_class_name_with_namespace" filename="plugin_shared_library_name">
  <mode name="emergency">
    <!-- Describe LED patterns here -->
  </mode>

  <mode name="idle">
    <!-- Describe LED patterns here -->
  </mode>
</plugin>
```

For the LED behaviors during each mode, we want the user to be able to define different configurations, patterns and animations. Therefore, we want the user to be able to define different "steps" of the mode animation each of which can have their own configurations of:

- Color: The RGBA color a particular LED will have during that step
- Intensity: Since LEDs are supposed to be a light source, it would be good to have an intensity setting that allows the user to create dimming/gradient-based animations.
- On Time: The duration for which this particular step stays "on".

Chaining up a bunch of steps with different settings for the above parameters would allow any user to create numerous animations for any kind of mode. In some scenarios, a user also might want to create a mode which stays on indefinitely with a particular setting. In such a scenario, it would be bad design to ask the user to enter a large value for the On Time. So, we can maybe have a boolean `always_on` attribute that informs the plugin that a step has to stay on indefinitely. Building upon this, we have an example snippet that looks something like:

```xml
<plugin name="plugin_class_name_with_namespace" filename="plugin_shared_library_name">
  <!-- Emergency mode that creates a Red blinking animation -->
  <mode name="emergency">
    <!-- First step that keeps the Red color on for 1 second -->
    <step always_on"false">
      <color>1 0 0 1</color>
      <intensity>3.0</intensity>
      <on_time>1.0</on_time>
    </step>
    <!-- First step that keeps the White color on for 1 second -->
    <step always_on"false">
      <color>1 1 1 1</color>
      <intensity>3.0</intensity>
      <on_time>1.0</on_time>
    </step>
  </mode>

  <!-- Charging mode which should indicate that the charging is on stably -->
  <mode name="charging">
    <!-- First step of the mode which sets green color for the LED indefinitely -->
    <step always_on"true">
      <color>0 1 0 1</color>
      <intensity>3.0</intensity>
    </step>
  </mode>
</plugin>
```

Now we actually want to define which entities in Gazebo would behave as our LEDs. The most obvious choice would be to use a Light entity. However, that alone is not enough. Why? See the gif below:

<!-- Show a gif that shows a blinking light in a sample world only with LED not visible -->

The gif above shows a a blinking light source on a simple robot model. Can you see the problem? Even though the blinking light causes illumination differences on surroundings, it is not enough as the actual LED model is itself not showing any differences. When you look at any real world indicators, they always have a diffuser on top to allow the light to spread out giving a softer uniform color.

<!-- Add gif that shows a real robot with blinking indicator -->

Currently in Gazebo there is no way we can simulate an actual diffuser. So what can we do? We can use the Visual entity of our LED model and have it change its material color in sync with the Light just like an actual diffuser. Therefore, we want our plugin to be able to use both, Light and Visual entity, as a LED. So maybe we can define LEDs in our plugin SDF as:

```xml
<plugin name="plugin_class_name_with_namespace" filename="plugin_shared_library_name">
  <led name="front_led">
    <visual_name>Name of the Visual</visual_name>
    <light_name>Name of the Light</light_name>
  </led>
  <led name="back_led">
    <visual_name>Name of the Visual</visual_name>
    <light_name>Name of the Light</light_name>
  </led>
</plugin>
```

For the name, we'll be using the [scoped name](https://gazebosim.org/api/sim/8/namespacegz_1_1sim.html#ab26a693e034871db72a7d3118d5233ac) of the entity so our plugin can easily find the actual entity that the user intends to use even if the multiple entities with the same name exists.

Finally, we want the user to be able to define a `<default_mode>`. It would also be good if the user is able to define a `<led_group_name>` which our plugin can use to advertise the `change_mode` service (`/led_group_name/change_mode`).

So this brings us to our final SDF snippet which looks like:

```xml
<plugin name="plugin_class_name_with_namespace" filename="plugin_shared_library_name">
  <led_group_name>some_name</led_group_name>

  <led name="front_led">
    <visual_name>Name of the Visual</visual_name>
    <light_name>Name of the Light</light_name>
  </led>
  <led name="back_led">
    <visual_name>Name of the Visual</visual_name>
    <light_name>Name of the Light</light_name>
  </led>

  <default_mode>some_name</default_mode>

  <!-- Emergency mode that creates a Red blinking animation -->
  <mode name="emergency">
    <!-- First step that keeps the Red color on for 1 second -->
    <step always_on"false">
      <color>1 0 0 1</color>
      <intensity>3.0</intensity>
      <on_time>1.0</on_time>
    </step>
    <!-- First step that keeps the White color on for 1 second -->
    <step always_on"false">
      <color>1 1 1 1</color>
      <intensity>3.0</intensity>
      <on_time>1.0</on_time>
    </step>
  </mode>

  <!-- Charging mode which should indicate that the charging is on stably -->
  <mode name="charging">
    <!-- First step of the mode which sets green color for the LED indefinitely -->
    <step always_on"true">
      <color>0 1 0 1</color>
      <intensity>3.0</intensity>
    </step>
  </mode>
</plugin>
```

Now this snippet will act as a nice blueprint for the development of our actual plugin.

#### A High Level DFD

In this step we will come with a high level DFD (Data Flow Diagram). This would help us in linking our SDF snippet with software-level components of Gazebo.

{{<figure src="high_level_dfd.png" width=800 loading="eager" align="center" caption="High Level DFD of the whole plugin.">}}

Let's briefly talk about each of the different components in our DFD above:

- SDF File: This is our world file which provides our world description containing the models and plugins
- Gazebo: This is, well, Gazebo which is the central part in this whole flow. It acts as the conductor in our Gazebo orchestra doing jobs such as parsing the SDF file, loading plugins, calling the plugin's interface functions, and running the simulation loop; basically everything.
- LedPlugin::Configure(): 
