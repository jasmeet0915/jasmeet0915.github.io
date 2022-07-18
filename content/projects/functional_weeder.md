---
title: "Functional Weeder"
date: 2019-12-23T20:56:42+06:00
type: project
weight: 2
image: "images/projects/functional_weeder.jpg"
category: ["Fleet Management | Elixir | Path Planning | 3D Printing"]
---

{{<figure src="/images/projects/functional_weeder.gif" alt="steward" loading="eager" width="720">}}

Functional Weeder (I am talking about the 'unwanted plants' in agricultural fields, just to be clear 😒) was a bot made during the Eyantra Robotics Competition of 2021-22. This challenge comprised of creating a fleet of agricultural robots that could autonomously perform tedious tasks like planting seeds & removing weeds. The software for the robot including all algorithms, hardware interface, communication with dashboard, etc. all had to written using Elixir which is a functional programming language.

> **Note:** To guy who spent most of his time with object oriented programming this was a BIG paradigm shift. Why you ask? Well, for starters, Elixir DOES not have loops since variables are immutable. Therefore people use recursions....(Yeah let that sink in for moment). 
I mean, there is a for loop per se, but it doesn't work we are used. Its basically just a function which runs a certain number of times and returns a value.

We wrote the following custom algorithms:
 * Centralized Fleet Management based on Auction-based Mechanism
 * Path Planning based on A* Algorithm considering things like distance, number of turns, etc. in cost function (Try imagining writing that with only recursions and no loops 😵‍💫)

Well our struggle didn't end here, after this we had write the hardware interface for a 2-DoF custom robotic arm and a Rack & Pinion based Linear Actuator seed loading mechanism. For this we used Elixir's [GenServers](https://hexdocs.pm/elixir/1.13/GenServer.html). 

All this was fine, but the most tedious and back-hurting (quite literally) job was to get the Line Following working. We wrote a basic PID based controller for the robot to allow it follow a white line on our flex arena. But the sensors were so sensitive and environment so noisy, that it would fail like most of the time. We tried many remedies. Some worked and improved the line following but not much. 
Still, we were able to reach the pre-finals among 5 Teams out 300 teams!

[Watch the Video](https://www.youtube.com/watch?v=RK3qnizgVEk) to see the project in action. 
Click to visit the [Client Side Repo](https://gitlab.com/insaanimanav/fw-clientrobota) or [Server Side Repo](https://gitlab.com/insaanimanav/fw-server)