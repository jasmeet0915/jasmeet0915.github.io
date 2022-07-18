---
title: "S.T.E.W.A.R.D"
date: 2019-12-23T20:56:42+06:00
type: project
weight: 1
image: "images/projects/steward.jpg"
category: ["ROS | OpenCV | Robot Perception | Grafana"]
---

{{<figure src="/images/projects/steward.gif" alt="steward" loading="eager" width="720">}}

One of the most unique & challenging problems I have worked on was during the [Rally-to-Tally hackathon by Mitsubishi America](https://www.hackerearth.com/challenges/hackathon/mitsubishi-americas-hackathon/). It was organized last year with an inventory management-based problem statement of counting a large number of industrial steel pipes with diameters ranging from 2 to 48 inches in an open warehouse spread over acres of land. 
Being from a robotics background, I knew that a robot would be the best candidate to do such a job with speed, accuracy & ease. Therefore, S.T.E.W.A.R.D was born.
> I know, I know. You are wondering what in the world S.T.E.W.A.R.D stands for? Well,... it stands for **S**mart **TE**rrestrial **WA**rehouse **R**obot **D**eployment.

So I teamed up with my brothers from different technological backgrounds & we started working on the project (well mostly :p). 
I started by setting up a simulation environment with life-size steel pipes in Gazebo and the ['Husky Rover'](https://clearpathrobotics.com/husky-unmanned-ground-vehicle-robot/) from Clearpath Robotics followed by working on the robot's software stack including a control loop using ROS & Python. During this, one of my teammates worked on making circle detection happen with `HoughCircles` using OpenCV & python. Once that was complete, Circle Detection was integrated into the robot's live camera feed using ROS & counting was implemented. 
While this was done, my other teammates were working on setting up `InfluxDB` & `Grafana` for a visualization dashboard to show Robot's health stats & mission statistics (counting, pipe diameter, date/time, etc). ROS Topics were linked with InfluxDB using a custom python node to visualize simulated data on the dashboard. 

{{<figure src="/images/projects/grafana_steward.png" alt="steward" loading="eager" width="720">}}

The project ended up winning the First Prize & teaching us a lot!

[Watch the Video](https://www.youtube.com/watch?v=jnn3Y15gVRA) to see the project in action. 
[Click Here](https://github.com/bhavyahoda/STEWARD-Rally-to-Tally) to visit the project repo.
