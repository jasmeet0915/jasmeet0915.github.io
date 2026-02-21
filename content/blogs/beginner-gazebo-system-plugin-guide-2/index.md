---
title: "A Beginner's Guide to Making System Plugins for Gazebo - Part 2: Making a LED Plugin "
weight: 2
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 2
---

{{< lead >}}
This is going to be a 2 part series. This part covers the basics of Gazebo, System Plugins, Plugin Interfaces, etc. The next part dives into the actual nitty-gritties of developing a plugin. If you are already familiar with the basics, feel free to skip this one and jump directly to the [next ↗](TODO: Add link to the next blog here)
{{< /lead >}}

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

[TODO: Brief intro here]

Let's start right where we left off in our Part 1!

## Making System Plugins for Gazebo

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
