---
title: "Beginner's Guide to Making System Plugins for Gazebo"
weight: 1
draft: false
description: "This blog is a beginner's guide to writing system plugins for Gazebo Robotics Simulator"
tags: ["blogs", "guide", "gazebo", "robotics", "system", "plugins", "led", "indicators", "simulation"]
series_order: 1
---

{{< lead >}}
This is going to be a long one folks! But I bet that at the end you'll have much more insights into Gazebo, it's architecture and how to add your own functionality to it via custom plugins while also learning a thing or two about programming along the way. I'll do my best to make it worth your time :)
{{< /lead >}}

Ever since I did GSoC with Gazebo, Open Robotics back in 2023 (read the official press release [here](https://www.openrobotics.org/blog/2023/6/14/2023-google-summer-of-code-students)), a majority portion of my DMs are full of students seeking guidance on how to contribute to Gazebo and/or participate in GSoC with Open Robotics. I have also met countless number of people interested in expanding Gazebo's functionality for their specific use cases whether its work, personal projects or just for the fun of it.

> **Note:** Whenever I say "Gazebo" in this blog, I am referring to the new Gazebo unless specified otherwise (Classic Gazebo).

Being someone (with limited access to good hardware) who started working on advanced robotics when the whole world was shutdown with quarantine, Gazebo was my only source to research, experiment and develop projects. Even some of my initial work experiences (internships) were completely remote and surrounded around developing robotic applications purely on simulation. Therefore, I did my best (and am still doing) to get the hang of simulation-based robotics and how Gazebo fits into that paradigm as one most widely used open source robotics simulator.

Based on my experience, the best way to achieve this, is to get into the nitty-gritties of how it all works. Thankfully, Gazebo's open source community and modular architecture makes it a bit easier (even though the documentation can be improved) if you know where to look. As I see it you have 2 starting positions:

- Look for the issues labelled "good first issue" in the [gz-sim repo](https://github.com/gazebosim/gz-sim/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22good%20first%20issue%22) and make an attempt at fixing them.
- Participate in Gazebo Tutorial Party which happens after every release (and yes you get free Gazebo t-shirts if you are on the leaderboard towards the end of of the Part. Not to brag, but I have 2 of them )
- Then move on 2 the bigger issues for bugs, enhancements, etc.

For me, [this](https://github.com/gazebosim/gz-sim/issues/1057) was the first ever issue I fixed in gazebo which allowed a user to have custom topics for the `JointStatePublisher` system plugin. This was followed by a couple of minor fixes, Gazebo Garden's Tutorial Party and [this](https://github.com/gazebosim/gz-sim/issues/1909) issue for porting the `LensFlare` plugin from Classic Gazebo to the new Gazebo. This became my segue into the journey of developing custom plugins and features for Gazebo.

With this blog, I aim to provide a detailed guide to writing system plugins for Gazebo by taking my newest plugin, `gz_sim_led_plugin`, as an example that lets you simulate LEDs/Indicators in Gazebo.

{{<figure src="gazebo_led_plugin_demo.gif" width=800 loading="eager" align="center" caption="Demo gif of the plugin in action showing 2 robots and an industrial tower lamp model each having different LED group with different modes.">}}

# Brief Introduction to Gazebo's Architecture and What really is a System Plugin
In this section, we are going to discuss the basic architecture and terminologies that you need as a precursor for making your own plugins

## First things first, What is Gazebo?
Let's start with the basics: What is Gazebo? well - duh, its a Robotics Simulator. But that is a user's perspective. What about from a developer's perspective? Well from a developer's perspective, Gazebo is a collection of [numerous C++ libraries](https://gazebosim.org/libs/) (16 to be precise) that work together to make every thing possible from physics, rendering, GUI, CLI, SDFormat parsing, etc. Libraries such as [`gz-physics`](https://github.com/gazebosim/gz-physics), [`gz-rendering`](https://github.com/gazebosim/gz-rendering) act as an abstraction that allows Gazebo to work with different physics and rendering engine making it hugely customizable. Libraries such as `gz-sim` and `gz-gui` act as the backend and frontend respectively, while `gz-transport` handles the communication between different sub systems of Gazebo using topics and services (much like ROS / ROS 2). I can go on but I hope you get the idea.


