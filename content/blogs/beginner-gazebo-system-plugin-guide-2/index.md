---
title: "A Beginner's Guide to Making System Plugins for Gazebo - Part 2: Making a LED Plugin "
weight: 2
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 2
---

{{< lead >}}
This is Part 2 of the series & assumes you have basic knowledge of Gazebo, System Plugins and Plugin Interfaces. If not, jump back to [Part 1 ↗](/blogs/beginner-gazebo-system-plugin-guide-1/).
{{< /lead >}}

{{< alert "lightbulb" >}}
**Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).
{{< /alert >}}

As mentioned in Part 1, I'll be taking my latest plugin, `gz_sim_led_plugin`, as a reference for all the explanations throughout this guide. 

Let's start right where we left off in our Part 1!

## Understanding Our Reference: The LED Plugin

{{< github repo="jasmeet0915/gz_sim_led_plugin" showThumbnail=false >}}

The idea behind this plugin is simple:

We want to **simulate LEDs/Indicators** in Gazebo. **Why?** Because all industrial systems always have some kind of indicators (mostly LEDs) that lets you know what's happening behind the scenes. **For instance,** a robot might

* Use **blinking red LEDs** in **fault / emergency** mode to tell operators that the robot is not functioning correctly and human intervention is required
* Use **blinking green LEDs** in **ready** mode to tell operators that the robot is *ready* to take missions.
* Use **solid yellow LEDs** when its **low on battery** to tell operators that the robot needs to be put on charge (or might go to auto charge, if available)
* Use **pulsing blue LEDs** for on-going **docking/undocking operation** to tell operators that the robot is performing precision maneuvers so caution is advised.

And so on.

{{<figure src="rm250_docking.gif" width=800 loading="eager" align="center" caption="RM250 Robot from Peer Robotics using blinking red LEDs while docking with a trolley. Do checkout their website: https://peerrobotics.ai/">}}

Naturally, such LED patterns are generally associated with different internal **modes** or **states** of a robot and as a robotics systems engineer, it might be a part of your job to ensure such **mode <-> LED mappings**. Therefore, like any piece of hardware, it makes sense to simulate LEDs for development and testing of such flows without the need of actual hardware.

{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

## Configuring our Plugin: ISystemConfigure

Before writing a plugin (or in fact any piece of software), it is essential to lay out how your potential users are going to use it. In our case, this will be through some configuration inside the `<plugin>` tags of an SDF file. This configuration is made available to our plugin as a `sdf::ElementConstPtr` instance through the `ISystemConfigure` interface. Therefore, it is essential to define our possible SDF snippet for the plugin so that it can act as a blueprint for our `Configure()` implementation.

### Describing our LED Modes: SDF to C++ Struct

Any potential user of our plugin would want to define a bunch of named `<modes>` with their distinct behavior for a bunch of LEDs. The SDF snippet for it would look somewhat like:

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

To define the LED animations/patterns during each mode, we should have different `<steps>` having their own configurations of:

- **Color:** RGBA color for the mode's active LEDs during that step.
- **Intensity:** Intensity of the LED light sourcem to allow users to create dimming/gradient-based animations.
- **On Time:** The duration for which this particular step stays "on".

Chaining up a bunch of `<step>` with different configurations would allow users to create numerous animations for any kind of `<mode>`. For `<mode>` with static configuration, we can have a boolean `always_on` attribute for `<step>`that keeps the step indefinitely "on".

Building upon this, we have our final mode description in SDF that looks something like:

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

**Structs in C++** are an obvious choice to represent our `<mode>` and `<step>` data inside the plugin:

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

### Describing our LEDs: Which Entities to use?

For describing our LEDs, we need to decide which entities in Gazebo would work best. The **most obvious choice** would be the [Light Entity](https://gazebosim.org/api/sim/9/classgz_1_1sim_1_1Light.html). **However, light alone is not enough.** Why? See the gif of a blinking light source in Gazebo below:

<!-- Show a gif that shows a blinking light in a sample world only with LED not visible -->

Can you see the problem? Even though the blinking light changes illumination on surroundings, it **does not truly represent an actual LED itself.** When you look at any real world LEDs, they always have a diffuser on top to spread the light for softer look (see the real robot gif above).

Currently, in Gazebo, we cannot simulate an actual diffuser. So what can we do? We can use the **Visual Entity** of our LED link and have it change its **Material** color in sync with the Light just like an actual diffuser. Therefore, we want our plugin to use both, Light and Visual entity, as an LED. So, LED descriptions in our plugin SDF would look like this:

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

And the equivalent C++ struct for the LED would be:

```C++
/// \brief Struct to define the individual LED
struct Led
{
  /// \brief Name of the LED
  std::string ledName{""};

  /// \brief Scoped name of the Visual of the LED
  std::string scopedVisualName{""};

  /// \brief Scoped name of the light of the LED
  std::string scopedLightName{""};

  /// \brief Entity of the visual to be used with this LED
  sim::Entity ledVisualEntity{kNullEntity};

  /// \brief Entity of the light to be used with this LED
  sim::Entity ledLightEntity{kNullEntity};

  /// \brief The default color to use when the LED is reset
  math::Color defaultColor{math::Color::White};

  /// \brief The default intensity to use for the light when LED is reset
  double defaultIntensity{0.0};
};
```

## Reading the SDF in Configure()

### Using the sdf::ElementConstPtr

Now that we have our SDF snippet figured out, we need to parse it in our `Configure()`. If you recall from Part 1, `Configure()` is the plugin interface which gets called by Gazebo once after loading the plugin. It gives us a `sdf::ElementConstPtr` instance as an agrument which is basically the root element of our `<plugin>` tag. From here, it's all about traversing the SDF tree. The `sdf::Element` API gives us a handful of methods for that.

Let's break down the key ones using snippets from our `Configure()` implementation:

#### HasElement()

Before reading any child element, you should check whether it actually exists. `HasElement()` takes a tag name as a string and returns `true` if a child with that name is present:

```C++
if (_sdf->HasElement("led_group_name"))
{
  this->dataPtr->ledGroupName = _sdf->Get<std::string>("led_group_name");
}
else
{
  gzwarn << "No name is specified for the led group, model name"
          << " will be used for the change_mode service name" << std::endl;
  this->dataPtr->ledGroupName = "led_" + this->dataPtr->model.Name(_ecm);
}
```

This pattern is used everywhere in a plugin.

For **optional** elements, it allows us to check if the user has provided a value or should we fall back to a sensible default. This is what happens in the snippet above where we set our `this->dataPtr->ledGroupName` from the optional `<led_group_name>` element if provided, else we default to use the model's name.

For **required** elements, say `<led>`, you'd typically log an error and return early to signal that the plugin can't proceed without it:

```C++
if (_sdf->HasElement("led"))
{
  // Parse LEDs...
}
else
{
  gzerr << "[LED PLUGIN] No LEDs have been defined "
        << "for the LED group: " << this->dataPtr->ledGroupName << std::endl;
  return;
}
```

{{< alert "lightbulb" >}}
**Note:** You'll notice `this->dataPtr->` used throughout the code. This is the [**PIMPL (Pointer to Implementation)**](https://en.cppreference.com/w/cpp/language/pimpl) idiom. A C++ pattern where the class holds a `std::unique_ptr` to a private struct containing all its member data. Nearly all Gazebo plugins and libraries use this pattern.
{{< /alert >}}

#### Get<T>()

Once you know an element exists, `Get<T>()` template function extracts its value and converts it to the type you specify. This function handles conversions for common types like `std::string`, `double`, `bool` and custom types defined in Gazebo like `gz::math::Color`, `gz::math::Vector`, etc. out of the box:

```C++
  this->dataPtr->ledGroupName = _sdf->Get<std::string>("led_group_name");
```

```C++
  math::Color ledColor = stepElem->Get<math::Color>("color");
```

#### FindElement()

`FindElement()` returns a `sdf::ElementConstPtr` pointing to the **first** child element matching the given tag name. This is your entry point whenever you need to iterate over multiple elements of the same name:

```C++
  sdf::ElementConstPtr ledElem = _sdf->FindElement("led");
  // ledElem now points to the first <led> child element
```

#### GetNextElement()

Once you have a pointer from `FindElement()`, calling `GetNextElement()` on it gives you the next element with the same name (similar to a linked list). It returns `nullptr` when there are no more matches, making it perfect for a `while` loop:

```C++
sdf::ElementConstPtr ledElem = _sdf->FindElement("led");
while (ledElem)
{
  // ... process this <led> element ...
  ledElem = ledElem->GetNextElement("led");
}
```

This pattern is used throughout this plugin for reading multiple `<led>`, `<mode>`, `<step>` description.

#### Factory Design Pattern is your friend

As your plugin's SDF configuration grows, dumping all the parsing logic into `Configure()` directly will turn it into a mess pretty quickly.

A much cleaner approach? **Static factory methods** called `FromSDF(sdf::ElementConstPtr)` on your data structs. Now each of your struct knows how to construct itself from an `sdf::ElementConstPtr`.

In our LED plugin, both, `Led` and `LedMode` structs, have a `fromSDF()` static factory method that takes the relevant `sdf::ElementConstPtr` and returns a `std::optional` instance of that struct. The `std::optional` let's the caller of the factory method know if the contruction was successful or not.

This keeps `Configure()` compact and focused on high-level orchestration rather than low-level parsing details:

```C++
// Read and create different LEDs as part of the group
if (_sdf->HasElement("led"))
{
  sdf::ElementConstPtr ledElem = _sdf->FindElement("led");

  while (ledElem)
  {
    auto led = Led::fromSDF(ledElem, _ecm);

    if (led.has_value())
    {
      this->dataPtr->allLedsInGroup.insert({led.value().ledName, led.value()});
    }

    ledElem = ledElem->GetNextElement("led");
  }
}
```

And the same pattern for modes:

```C++
// Read the described LED modes
if (_sdf->HasElement("mode"))
{
  sdf::ElementConstPtr ledModesElem = _sdf->FindElement("mode");

  while (ledModesElem)
  {
    auto ledMode = LedMode::fromSDF(ledModesElem);

    if (ledMode.has_value())
    {
      this->dataPtr->allLedModes.push_back(ledMode.value());
    }

    ledModesElem = ledModesElem->GetNextElement("mode");
  }
}
```

Here's what the `LedMode::fromSDF()` factory method looks like on the inside. Notice how it uses the exact same `HasElement()`, `Get()`, `FindElement()`, and `GetNextElement()` methods on the `sdf::ElementConstPtr` instance of our root `<mode>` element:

```C++
/// \brief Static factory method to create an instance of LedMode
/// using the sdf element
/// \param[in] _sdf SDF Element from which you want to construct
/// the LED Mode
public: static std::optional<LedMode> fromSDF(sdf::ElementConstPtr _sdf)
{
  LedMode ledMode;

  // Read and set the name of the LED Mode
  if (!_sdf->HasAttribute("name"))
  {
    gzerr << "[LED PLUGIN] [LED MODE] Name attribute is "
          << "missing. Can't construct led mode" << std::endl;

    return std::nullopt;
  }

  ledMode.name = _sdf->Get<std::string>("name");
  gzmsg << "Adding LED Mode: " << ledMode.name << std::endl;

  // Read the active LEDs for this mode if any
  if (_sdf->HasElement("active_leds"))
  {
    sdf::ElementConstPtr activeLedsElem = _sdf->FindElement("active_leds");

    if (activeLedsElem->HasElement("led"))
    {
      sdf::ElementConstPtr ledElem = activeLedsElem->FindElement("led");

      while (ledElem)
      {
        std::string ledName = ledElem->Get<std::string>();
        ledMode.activeLedNames.push_back(ledName);

        gzmsg << "[LED PLUGIN][LED MODE] Mode [" << ledMode.name
              << "] uses LED: " << ledName << std::endl;

        ledElem = ledElem->GetNextElement("led");
      }
    }
  }

  // Read the different steps involved in this LED Mode
  if (!_sdf->HasElement("step"))
  {
    gzerr << "[LED PLUGIN] [LED MODE] No steps are given for "
          << " the LED mode: " << ledMode.name << ". Can't "
          << "construct led mode" << std::endl;

    return std::nullopt;
  }

  sdf::ElementConstPtr stepElem = _sdf->FindElement("step");
  while (stepElem)
  {
    LedModeStep ledModeStep;

    if (stepElem->HasAttribute("always_on"))
    {
      ledModeStep.alwaysOn = stepElem->Get<bool>("always_on");
    }

    if (stepElem->HasElement("color"))
    {
      math::Color ledColor = stepElem->Get<math::Color>("color");
      ledModeStep.ledColor = ledColor;
    }

    if (stepElem->HasElement("on_time"))
    {
      if (!ledModeStep.alwaysOn)
      {
        ledModeStep.ledOnTime = std::chrono::duration<double>(
          stepElem->Get<double>("on_time"));
      }
      else
      {
        gzwarn << "LED Mode: " << ledMode.name << " step is set to "
                << "always on, on_time will be ignored" << std::endl;
      }
    }

    if (stepElem->HasElement("intensity"))
    {
      ledModeStep.lightIntensity = stepElem->Get<double>("intensity");
    }

    ledMode.modeSequenceSteps.push_back(ledModeStep);
    stepElem = stepElem->GetNextElement("step");
  }

  return ledMode;
}
```

#### Getting our Visual & Light Entity IDs

Parsing names from the SDF is only half the story. To actually change a Visual's material or a Light's color at runtime, we need their **Entity IDs**. Through the `Led::fromSDF()` factory method, we resolve the scoped names from our `<led>` description into Entity IDs that we can use later in `PreUpdate()`.

This is where [`gz::sim::entitiesFromScopedName()`](https://gazebosim.org/api/sim/8/namespacegz_1_1sim.html#ab26a693e034871db72a7d3118d5233ac) comes in handy. It takes a scoped name string (like `my_robot_model::led_link::led_visual`) and the `gz::sim::EntityComponentManager`, and returns a `std::unordered_set<Entity>` of all entities matching that name. Here's the full `Led::fromSDF()` factory method handling the SDF parsing and entity resolution:

```C++
public: static std::optional<Led> fromSDF(sdf::ElementConstPtr _sdf,
  const EntityComponentManager &_ecm)
{
  Led led;

  if (!_sdf->HasAttribute("name"))
  {
    gzerr << "[LED PLUGIN][LED] Name attribute for the LED is "
          << "missing. Can't construct LED" << std::endl;
    return std::nullopt;
  }

  led.ledName = _sdf->Get<std::string>("name");
  gzmsg << "[LED PLUGIN][LED] Creating led: " << led.ledName << std::endl;

  // Use the visual name to find the Visual entity provided for LED
  if (_sdf->HasElement("visual_name"))
  {
    led.scopedVisualName = _sdf->Get<std::string>("visual_name");
    std::unordered_set<Entity> visualEntities = gz::sim::entitiesFromScopedName(led.scopedVisualName, _ecm);
    gzmsg << "Found " << visualEntities.size() << " entities for the visual named: "
          << led.scopedVisualName << std::endl;

    if (visualEntities.empty())
    {
      gzerr << "[LED PLUGIN][LED] No visuals found with the name: "
            << led.scopedVisualName << ". Can't use visuals for the LED" << std::endl;
    }
    else
    {
      if (visualEntities.size() > 1)
      {
        gzerr << "[LED PLUGIN][LED] Multiple visuals found with the name: "
              << led.scopedVisualName << ". Using the first one found" << std::endl;
      }

      led.ledVisualEntity = *visualEntities.begin();
    }
  }

  // Use the light name to find the Light entity provided for LED
  if (_sdf->HasElement("light_name"))
  {
    led.scopedLightName = _sdf->Get<std::string>("light_name");
    std::unordered_set<Entity> lightEntities = gz::sim::entitiesFromScopedName(led.scopedLightName, _ecm);

    gzmsg << "Found " << lightEntities.size() << " entities for the light named: "
          << led.scopedLightName << std::endl;

    if (lightEntities.empty())
    {
      gzerr << "[LED PLUGIN][LED] No lights found with the name: "
            << led.scopedLightName << ". Can't use lights for the LED" << std::endl;
    }
    else
    {
      if (lightEntities.size() > 1)
      {
        gzerr << "[LED PLUGIN][LED] Multiple lights found with the name: "
              << led.scopedLightName << ". Using the first one found" << std::endl;
      }

      led.ledLightEntity = *lightEntities.begin();
    }
  }

  // Read the default state of the LED if provided
  if (_sdf->HasElement("default_state"))
  {
    sdf::ElementConstPtr defaultStateElem = _sdf->FindElement("default_state");

    if (defaultStateElem->HasElement("color"))
    {
      led.defaultColor = defaultStateElem->Get<math::Color>("color");
    }

    if (defaultStateElem->HasElement("intensity"))
    {
      led.defaultIntensity = defaultStateElem->Get<double>("intensity");
    }
  }

  return led;
}
```

## Applying the LED Modes: ISystemPreUpdate

This is where the magic happens. The `PreUpdate()` method runs **before every physics step** in the simulation loop. For our LED plugin, this is the perfect place to update the LED appearance based on the current mode and its steps.

```C++
// Set the current step from the current LED Mode
LedModeStep currentLedModeStep = this->dataPtr->currentLedMode.modeSequenceSteps[this->dataPtr->currentModeStepIdx];

// Change the visual and light properties of the LEDs based on the current mode
if (this->dataPtr->currentLedMode.activeLedNames.empty())
{
  // Set the state of all LEDs in the group according to the current LED Mode
  for (const auto &led : this->dataPtr->allLedsInGroup)
  {
    // Set the visual properties if the visual entity is not null
    if (led.second.ledVisualEntity != kNullEntity)
    {
      this->dataPtr->SetVisualProperties(led.second.ledVisualEntity, _ecm, currentLedModeStep.ledColor);
    }

    // Set the light properties if the light entity is not null
    if (led.second.ledLightEntity != kNullEntity)
    {
      this->dataPtr->SetLightProperties(led.second.ledLightEntity, _ecm, currentLedModeStep.ledColor,
        currentLedModeStep.lightIntensity);
    }
  }
}
else
{
  // Set the state of the LEDs in the activeLedNames for the current LED Mode
  for (const std::string &ledName : this->dataPtr->currentLedMode.activeLedNames)
  {
    // Set the visual properties if the visual entity is not null
    if (this->dataPtr->allLedsInGroup[ledName].ledVisualEntity != kNullEntity)
    {
      this->dataPtr->SetVisualProperties(this->dataPtr->allLedsInGroup[ledName].ledVisualEntity,
        _ecm, currentLedModeStep.ledColor);
    }

    // Set the light properties if the light entity is not null
    if (this->dataPtr->allLedsInGroup[ledName].ledLightEntity != kNullEntity)
    {
      this->dataPtr->SetLightProperties(this->dataPtr->allLedsInGroup[ledName].ledLightEntity,
        _ecm, currentLedModeStep.ledColor, currentLedModeStep.lightIntensity);
    }
  }
}
```

### Using UpdateInfo for Timed Operations

Our plugin's `PreUpdate()` callback receives an `UpdateInfo` struct that contains the current simulation time. This is how our plugin keeps track of the "on_time" of a step and knows when to advance to the next.

For steps with `alwaysOn` set to `true`, it's simple - apply the step once and do nothing else:

```C++
// If this step is supposed to be always on then just return
if (currentLedModeStep.alwaysOn)
{
  return;
}
```

For animated sequences, we track a `cycleStartTime` and compare it against the current simulation time to figure out how long the current step has been active. When the elapsed time exceeds the step's `ledOnTime`, we advance to the next step (cycling back to the beginning when we hit the end):

```C++
// Set the cycle start time
if (this->dataPtr->cycleStartTime == std::chrono::duration<double>::zero() ||
    this->dataPtr->cycleStartTime > this->dataPtr->currentSimTime)
{
  this->dataPtr->cycleStartTime = this->dataPtr->currentSimTime;
}
std::chrono::duration<double> elapsed = this->dataPtr->currentSimTime - this->dataPtr->cycleStartTime;

// If we have crossed the elapsed time of the step on time, move to the next step
if (elapsed > currentLedModeStep.ledOnTime)
{
  this->dataPtr->currentModeStepIdx++;

  // If we have reached the end of the steps, cycle back to the first one
  if (this->dataPtr->currentModeStepIdx == this->dataPtr->currentLedMode.modeSequenceSteps.size())
  {
    this->dataPtr->currentModeStepIdx = 0;
  }

  // Reset the cycle start time
  this->dataPtr->cycleStartTime = std::chrono::duration<double>::zero();
}
```

The extra check `cycleStartTime > currentSimTime` handles the case where the simulation is reset — sim time jumps back to zero, so we need to re-initialize the cycle start.

### VisualCmd and LightCmd Components

So how do we actually tell the rendering engine to change a Visual's color or a Light's intensity? We don't call rendering APIs directly. Instead, we use our entity's **command components**.

Specifically:
- **`components::VisualCmd`** — Attach this to a Visual entity (if not already attached) to request changes to its material. It takes a `gz::msgs::Visual` protobuf message.
- **`components::LightCmd`** — Attach this to a Light entity (if not already attached) to change its properties. It takes a `gz::msgs::Light` protobuf message.

Here's how we use them in our plugin's `SetVisualProperties()` & `SetLightProperties()` functions:

```C++
// Update the Visual's material color
msgs::Visual visualMsg;
msgs::Material *materialMsg = visualMsg.mutable_material();
msgs::Set(materialMsg->mutable_diffuse(), ledColor);
msgs::Set(materialMsg->mutable_emissive(), ledColor);

auto visualCmdComp = _ecm.Component<components::VisualCmd>(_visualEntity);
if (!visualCmdComp)
{
  _ecm.CreateComponent(_visualEntity, components::VisualCmd(visualMsg));
}
else
{
  auto state = visualCmdComp->SetData(visualMsg, visualEq) ?
    ComponentState::OneTimeChange : ComponentState::NoChange;
  _ecm.SetChanged(_visualEntity, components::VisualCmd::typeId, state);
}

// Update the Light's color and intensity
msgs::Light lightMsg;
msgs::Set(lightMsg.mutable_diffuse(), ledColor);
lightMsg.set_intensity(lightIntensity);

auto lightCmdComp = _ecm.Component<components::LightCmd>(_lightEntity);
if (!lightCmdComp)
{
  _ecm.CreateComponent(_lightEntity, components::LightCmd(lightMsg));
}
else
{
  auto state = lightCmdComp->SetData(lightMsg, lightEq) ?
    ComponentState::OneTimeChange : ComponentState::NoChange;
  _ecm.SetChanged(_lightEntity, components::LightCmd::typeId, state);
}
```

The idea is: you populate a protobuf message with the desired state, then either create or update the corresponding `Cmd` component on the entity. Then, its upto Gazebo's internals to pick these command components and apply the changes.

### Advertising the Mode Change Service

So we can define modes. We can animate LEDs. But how does the rest of the system (like your robot's ROS stack) actually **tell** our plugin to switch modes at runtime?

Enter **Gazebo Transport Services**. In `Configure()`, we advertise a service that allows external code to request a mode change. The service name uses the `<led_group_name>` we parsed earlier, giving us something like `/tower_lamp_leds/change_mode`.

```C++
// In Configure()
std::string serviceName = "/" + this->ledGroupName + "/change_mode";
this->node.Advertise(serviceName, &LedPlugin::OnChangeModeRequest, this);
```

The callback is straightforward. It receives the requested mode name and finds it our `allLedModes` vector. If found, the callback udpates the `currentLedMode` & resets the `currentModeStepIdx` and `cycleStartTime`:

```C++
bool LedPluginPrivate::OnLedModeChange(const msgs::StringMsg &_req,
  msgs::Boolean &_resp)
{
  std::lock_guard<std::mutex> lock(this->mutex);

  std::string requestedModeName = _req.data();
  gzmsg << "[LED PLUGIN] [ON MODE CHANGE] received request to change mode to: "
        << requestedModeName << std::endl;

  auto ledModeIter = std::find_if(this->allLedModes.begin(), this->allLedModes.end(),
    [&](LedMode _mode)
    {
      if (_mode.name == requestedModeName)
      {
        return true;
      }
      return false;
    });

  // If the requested mode was not found
  if (ledModeIter == this->allLedModes.end())
  {
    gzerr << "[LED PLUGIN] Requested LED Mode: " << requestedModeName
          << " was not described" << std::endl;

    _resp.set_data(false);
    return false;
  }

  gzmsg << "[LED PLUGIN] [ON MODE CHANGE] Changing led mode from: "
        << this->currentLedMode.name << " to: " << requestedModeName << std::endl;

  this->currentLedMode = *(ledModeIter);
  this->currentModeStepIdx = 0;
  this->cycleStartTime = std::chrono::duration<double>::zero();
  gzmsg << "[LED PLUGIN] [ON MODE CHANGE] Current led mode set to: "
        << this->currentLedMode.name << std::endl;

  _resp.set_data(true);

  // Setting LEDs as not ready as we want them to be reset after a mode change
  // This is done because we cannot directly reset the LEDs from here as ECM is required
  this->ledsReady = false;
  return true;
}
```

You can now trigger mode changes from the command line using `gz service` or from your ROS stack using the `ros_gz_bridge`.

### Ending Notes

And that's pretty much the whole picture! To quickly recap what we covered in this part: we designed our plugin's SDF configuration, parsed it in `Configure()`, used `VisualCmd` and `LightCmd` components in `PreUpdate()` to animate LEDs, and finally advertised a Gazebo Transport service to allow runtime mode switching.

One piece of advice: the **API reference is your friend**. Seriously. Gazebo's API docs might not win any beauty contests, but they are comprehensive. Whenever you're stuck wondering "how do I change X on entity Y?", chances are there's a component or a helper function for it. The [gz-sim API reference](https://gazebosim.org/api/sim/8/) and the [gz-msgs reference](https://gazebosim.org/api/msgs/10/) are especially useful when working with command components.
