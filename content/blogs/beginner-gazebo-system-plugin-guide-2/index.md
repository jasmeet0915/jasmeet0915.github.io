---
title: "A Beginner's Guide to Making System Plugins for Gazebo - Part 2: Making a LED Plugin "
weight: 2
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 2
---

{{< lead >}}
This is part 2 of the series and assumes you have basic knowledge of Gazebo, System Plugins and Plugin Interfaces. If not, jump back to [part 1 ↗](blogs/beginner-gazebo-system-plugin-guide-1).
{{< /lead >}}

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

If you are new to this series, let's do a quick recap:
As mentioned before, I'll be taking my latest plugin, `gz_sim_led_plugin`, as a reference for all the explanations throughout this guide. 
As far as plugins go, this is more towards the beginner-friendly side and I believe having a working example that allows you to see everything in action, helps in better understanding.

Let's start right where we left off in our Part 1!

## Understanding Our Reference: The LED Plugin

{{< github repo="jasmeet0915/gz_sim_led_plugin" showThumbnail=false >}}

The idea behind this plugin is simple: we want to simulate LEDs/Indicators in Gazebo. Why? Because all industrial systems always have some kind of indicators (mostly LEDs) that tells the surrounding people what's happening behind the scenes. **For instance,** a robot might

* Use **blinking red LEDs** in **fault / emergency** mode. This lets operators know that the robot might not function correctly and human intervention is required
* Use **blinking green LEDs** in **ready** mode. This tells operators that the robot can now take new missions.
* Use **solid yellow LEDs** when its **low on battery**. This tells operators that the robot needs to be put on charge (or might go to auto charge, if available)
* Use **pulsing blue LEDs** for on-going **docking/undocking operation**. This tells operators that the robot is performing precision maneuvers so caution is advised.

And so on. I hope you get the idea.

{{<figure src="rm250_docking.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

One common pattern we see from the above examples is that the different LED patterns are generally associated with different **modes** or **states** of a robot.
As a systems engineer in robotics, it would probably be a part of your job to make sure that you map such different modes of the robot to different patterns / configuration of these indicators. Therefore, like any piece of hardware, simulating LEDs provides the benefit of rapid prototyping, development and testing of such flows without the need of actual hardware.

{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="RM250 Robot from Peer Robotics using blinking red LEDs while docking with a trolley. Do checkout their website: https://peerrobotics.ai/">}}

## Making System Plugins for Gazebo

### Configuring our Plugin: ISystemConfigure

Before writing a plugin (or in fact any piece of software), it is essential to lay out how your potential users are going to use it. In our case, this will be through some configuration inside the `<plugin>` tags of an SDF file. This configuration is the first thing made available to our plugin in the form of a `sdf::ElementConstPtr` instance through the `ISystemConfigure` interface. It is essential that we define our possible SDF snippet for the plugin so that it can act as a blueprint for our `Configure()` implementation. Therefore, in this section we will think about what a user of our plugin might need to configure, come with a friendly SDF snippet for it and then parallely see how it is implemented

#### Describing our LED Modes: SDF to C++ Struct

As we have already established, different LED patterns/configurations are generally mapped to different modes/behaviours. Therefore, a potential user of our plugin would want to define a bunch of named `<modes>` each of which describes their own behaviour for a bunch of LEDs. This would look somewhat like:

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

For the LED behaviors during each mode, user would want to define different patterns or animations. These animations would have different `<steps>` having their own configurations of:

- **Color:** The RGBA color for the mode's active LEDs during that step
- **Intensity:** Since LEDs are supposed to be a light source, it would be good to have an intensity setting that allows the user to create dimming/gradient-based animations.
- **On Time:** The duration for which this particular step stays "on".

Chaining up a bunch of `<step>` with different configurations would allow any user to create numerous animations for any kind of `<mode>`. For `<mode>` with static LED colors, we can have a boolean `always_on` attribute for `<step>`that informs the plugin if a step has to stay on indefinitely.

Building upon this, we have an example SDF snippet that looks something like:

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

Structs in C++ are an obvious choice to represent our `<mode>` and `<step>` inside the plugin:

```C++
/// \brief Struct to define the LED Mode
struct LedMode
{
  /// \brief Name of the LED Mode
  std::string name{""};

  /// \brief List of string names of the active LEDs in this mode
  std::vector<std::string> activeLedNames;

  /// \brief A list steps in sequence for the LED mode
  std::vector<LedModeStep> modeSequenceSteps;
};

/// \brief Struct to define a step in the LED Mode Sequence
struct LedModeStep
{
  /// \brief Whether the LED should stay on indefinitely
  bool alwaysOn{false};

  /// \brief Whether the LED should stay on indefinitely
  math::Color ledColor{math::Color::White};

  /// \brief Seconds for which the LED should stay on.
  /// Used when alwaysOn is false
  std::chrono::duration<double> ledOnTime{1s};

  /// \brief Intensity for any lights as part of the LED
  /// Defaults to 1.0
  double lightIntensity{1.0};
};
```

#### Describing our LEDs: Which Entities to use?

For describing our LEDs, first we should decide which entities in Gazebo would best represent an LED. The most obvious choice would be the **Light** entity. However, that alone is not enough. Why? See the gif below:

<!-- Show a gif that shows a blinking light in a sample world only with LED not visible -->

This gif shows a blinking light source on a simple robot model. Can you see the problem? Even though the blinking light changes illumination on surroundings, it is not enough to represent an actual LED model itself. When you look at any real world indicators, they always have a diffuser on top to allow the light to spread out giving a softer uniform color.

Currently in Gazebo there is no way we can simulate an actual diffuser. So what can we do? We can use the **Visual** entity of our LED model and have it change its **Material** color in sync with the Light just like an actual diffuser. Therefore, we want our plugin to use both, Light and Visual entity, as an LED. So our LEDs description in our plugin SDF would look like this:

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

#### Scoped Names in Gazebo
For the name, we'll be using the [scoped name](https://gazebosim.org/api/sim/8/namespacegz_1_1sim.html#ab26a693e034871db72a7d3118d5233ac) of the entity so our plugin can easily find the actual entity that the user intends to use even if the multiple entities with the same name exists.

### Reading the SDF in Configure()

Now that we have our SDF snippet figured out, we need to actually parse it in the `Configure()` method. If you recall from Part 1, `Configure()` gives us a `sdf::ElementConstPtr` which is basically the root element of our `<plugin>` tag. From here, it's all about traversing the SDF tree — think of it like parsing XML (because, well, it is XML).

Here's the gist of how you'd read the modes and LEDs:

```C++
void LedPlugin::Configure(
    const Entity &_entity,
    const std::shared_ptr<const sdf::Element> &_sdf,
    EntityComponentManager &_ecm,
    EventManager &_eventMgr)
{
  // Read the LED group name
  if (_sdf->HasElement("led_group_name"))
    this->ledGroupName = _sdf->Get<std::string>("led_group_name");

  // Parse LED definitions
  auto ledElem = _sdf->FindElement("led");
  while (ledElem)
  {
    std::string ledName = ledElem->Get<std::string>("name");
    std::string visualName = ledElem->Get<std::string>("visual_name");
    std::string lightName = ledElem->Get<std::string>("light_name");
    // ... store these in your data structures
    ledElem = ledElem->GetNextElement("led");
  }

  // Parse Modes
  auto modeElem = _sdf->FindElement("mode");
  while (modeElem)
  {
    LedMode mode;
    mode.name = modeElem->Get<std::string>("name");

    // Parse steps within each mode
    auto stepElem = modeElem->FindElement("step");
    while (stepElem)
    {
      LedModeStep step;
      step.alwaysOn = stepElem->Get<bool>("always_on");
      step.ledColor = stepElem->Get<math::Color>("color");
      if (stepElem->HasElement("intensity"))
        step.lightIntensity = stepElem->Get<double>("intensity");
      if (stepElem->HasElement("on_time"))
        step.ledOnTime = std::chrono::duration<double>(
            stepElem->Get<double>("on_time"));

      mode.modeSequenceSteps.push_back(step);
      stepElem = stepElem->GetNextElement("step");
    }
    // ... store the mode
    modeElem = modeElem->GetNextElement("mode");
  }
}
```

The pattern is pretty straightforward: check if an element exists, grab its value with the appropriate type, and move to the next sibling using `GetNextElement()`. The `sdf::Element` API is your best friend here — it handles type conversions for common types like `math::Color`, `std::string`, `double`, etc. out of the box.

One thing to note: we also use `Configure()` to look up the actual **Entity IDs** for our visuals and lights using the scoped names we parsed. You can use helper functions from `gz::sim` like `EntityFromScopedName()` or query the `EntityComponentManager` directly to find entities matching the names provided in the SDF.

### Applying the LED Modes: ISystemPreUpdate

This is where the magic happens. The `PreUpdate()` method runs **before every physics step** in the simulation loop. For our LED plugin, this is the perfect place to update the LED appearance based on the current mode and its animation sequence.

The basic flow inside `PreUpdate()` looks like:

1. **Check the current mode** and figure out which step in the mode's sequence we should be on (based on elapsed time).
2. **Get the color and intensity** for the current step.
3. **Apply the changes** to both the Visual (material color) and the Light (diffuse color + intensity) entities.

For steps with `alwaysOn` set to `true`, we just set it once and forget about it. For animated sequences, we keep track of elapsed time and cycle through the steps. When the `on_time` for a step is up, we move to the next step — and when we reach the end, we loop back to the beginning. Simple as that.

#### VisualCmd and LightCmd Components

Gazebo uses an **Entity Component System (ECS)** architecture (as we covered in Part 1). So how do we actually tell the rendering engine to change a Visual's color or a Light's intensity? We don't call rendering APIs directly — instead, we use **command components**.

Specifically:
- **`components::VisualCmd`** — Attach this to a Visual entity to request changes to its material (like diffuse/emissive color, specular, etc.). Under the hood, it takes a `msgs::Visual` protobuf message.
- **`components::LightCmd`** — Attach this to a Light entity to change its properties (color, intensity, range, etc.). This one takes a `msgs::Light` protobuf message.

Here's roughly what it looks like in practice:

```C++
// Update the Visual's material color
msgs::Visual visualMsg;
msgs::Material *materialMsg = visualMsg.mutable_material();
msgs::Set(materialMsg->mutable_diffuse(), ledColor);
msgs::Set(materialMsg->mutable_emissive(), ledColor);

auto visualCmdComp = _ecm.Component<components::VisualCmd>(visualEntity);
if (visualCmdComp)
  visualCmdComp->SetData(visualMsg);  // Update existing
else
  _ecm.CreateComponent(visualEntity, components::VisualCmd(visualMsg));  // Create new

// Update the Light's color and intensity
msgs::Light lightMsg;
msgs::Set(lightMsg.mutable_diffuse(), ledColor);
lightMsg.set_intensity(lightIntensity);

auto lightCmdComp = _ecm.Component<components::LightCmd>(lightEntity);
if (lightCmdComp)
  lightCmdComp->SetData(lightMsg);
else
  _ecm.CreateComponent(lightEntity, components::LightCmd(lightMsg));
```

The idea is: you populate a protobuf message with the desired state, then either create or update the corresponding `Cmd` component on the entity. Gazebo's internal systems pick up these command components and apply the changes to the actual rendering scene. It's a clean pattern — your plugin never touches the renderer directly.

### Advertising the Mode Change Service

So we can define modes. We can animate LEDs. But how does the rest of the system (like your robot's ROS stack) actually **tell** our plugin to switch modes at runtime?

Enter **Gazebo Transport services**. In `Configure()`, we advertise a service that allows external code to request a mode change. The service name uses the `<led_group_name>` we parsed earlier, giving us something like `/front_leds/change_mode`.

```C++
// In Configure()
std::string serviceName = "/" + this->ledGroupName + "/change_mode";
this->node.Advertise(serviceName, &LedPlugin::OnChangeModeRequest, this);
```

The callback is straightforward — it receives the requested mode name, validates it against our list of known modes, and switches the current mode:

```C++
bool LedPlugin::OnChangeModeRequest(
    const msgs::StringMsg &_req, msgs::Boolean &_res)
{
  std::string requestedMode = _req.data();
  if (this->modes.find(requestedMode) != this->modes.end())
  {
    this->currentMode = requestedMode;
    _res.set_data(true);
  }
  else
  {
    gzwarn << "Requested mode '" << requestedMode << "' not found!" << std::endl;
    _res.set_data(false);
  }
  return true;
}
```

You can then trigger mode changes from the command line using `gz service` or from your ROS nodes through the `ros_gz_bridge`. Pretty handy for testing — you don't need to wire up your entire ROS stack just to see if the LEDs blink correctly.

### Ending Notes

And that's pretty much the whole picture! To quickly recap what we covered in this part: we designed our plugin's SDF configuration, parsed it in `Configure()`, used `VisualCmd` and `LightCmd` components in `PreUpdate()` to animate LEDs, and finally advertised a Gazebo Transport service to allow runtime mode switching.

One piece of advice: the **API reference is your friend**. Seriously. Gazebo's API docs might not win any beauty contests, but they are comprehensive. Whenever you're stuck wondering "how do I change X on entity Y?", chances are there's a component or a helper function for it — you just need to look it up. The [gz-sim API reference](https://gazebosim.org/api/sim/8/) and the [gz-msgs reference](https://gazebosim.org/api/msgs/10/) are especially useful when working with command components.
