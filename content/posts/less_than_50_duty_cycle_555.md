---
title: "Getting Less than 50% Duty Cycle with 555 Timer in Astable Mode"
date: 2022-07-18T:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["first"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

A couple months back, I decided to dive into some basic electronics and where better to start than the beloved [555 Timer IC](https://www.ti.com/lit/ds/symlink/lm555.pdf)  ¯\\\_(ツ)_/¯. Being a strong believer of project-based learning, I decided to make a project using the timer instead of just cramming up the theory. Fast forwarding a couple weeks, here I present to you **555 Timer based Servo Tester 🎉** 

{{<figure src="/images/blogs/555_timer_duty_cycle/servo_tester.gif" loading="eager" width="400" caption="555 Timer Servo Tester in action. Here I am controlling the base joint servo of a robotic arm & drive servo for a linear actuator mechanism on one of my mobile robots.">}}

In this blog, I would like to share some interesting problems I faced and their solutions. But first, let's start with the basics.

# How Servo Motors Work 

So, a servo motor is basically a motor which can be moved to a particular angle. Internally, the servo uses a closed loop system to control the position of the shaft. Like any other closed loop system, this system also needs a feedback (position feedback in this case) to control the output. This is achieved with the help of a potentiometer. 

But how dow you send angles as commands to the servo? 

Well, a servo basically has 3 wires 2 of which are obviously VCC and GND for powering the internal DC motor. The third wire is where the magic happens. The third wire is the signal wire where you provide a **PWM (Pulse Width Modulated)** signal to get different angles. The relation between PWM signal and servo angle can be seen in the table below.

| Servo Angle (degrees) | Signal On Time (ms) | Period (ms) | Duty Cycle (%) |   |
|-----------------------|---------------------|-------------|----------------|---|
| 0                     | 1                   | 20          | 5              |   |
| 90                    | 1.5                 | 20          | 7.5            |   |
| 180                   | 2                   | 20          | 10             |   |

> **Note:** The above values are all ideal. In reality, I have even noticed the duty cycle for some servos to range from 2% to 14%!

So from this information we can conclude that to move a servo all we need is some power and a PWM signal. So how do we do that? We use the 555 Timer 😎 

> **Note:** We could also use a controller or a development board like arduino for this. Heck they even have in-built [libraries](https://www.arduino.cc/reference/en/libraries/servo/) so you don't even have to write the code that generates the PWM signal according to the above table. You just pass the angle to the `servo.write()` fucntion and sit back while the library does its magic. But, where's the fun in that :p

# 555 Timer and its Different Modes

The 555 Timer IC is a cheap and precise timing IC that can be used to generate various kinds of clock or PWM signals. The 555 timer can be used in following modes:
 * **Monostable Mode:** In this mode, the output of the 555 timer is driven high for a fixed time when an input trigger is detected. This mode, therefore, contains a single stable output state - hence the name 'Monostable'. Know more about this mode [here](https://www.youtube.com/watch?v=81BgFhm2vz8)

 {{<figure src="/images/blogs/555_timer_duty_cycle/monostable.gif" loading="eager" width="720" caption="555 Timer in Monostable Mode. The output turns ON for about 1.1s when switch is closed & Trig is pulled to GND. While the output is ON it completely ignores the Trig input.">}}

 * **Bistable Mode:** In this mode, the output of the 555 timer is driven high when the input trigger is detected and driven low when reset is driven low. This mode, therefore, contains 2 stable output states - hence the name 'Bistable'. Know more about this mode [here](https://www.youtube.com/watch?v=WCwJNnx36Rk)
 * **Astable Mode:** In this mode, the output of the 555 timer generates a square wave without any intervention from the user. The square wave is infact a PWM signal whose duty cycle can be adjusted by changing the values of the timing capacitors and resistors. Know more about this mode [here](https://www.youtube.com/watch?v=kRlSFm519Bo)

>**Note:** 'Stable' here simply means that the output will tend to stay in that state until an input trigger is detected. In 'Monostable' there is a single stable state which is the default state of the output and the output can be taken out of that state with the input trigger. In case of 'Bistable', both states of the output are considered stable and you can switch between them using 2 different input triggers. The 'Astable' mode however, does not have any stable state and just gives a square wave on the output.

# Operating Servos with 555 timer in Astable Mode
Since the **Astable** mode of operation provides us with a PWM signal at the output, its pretty obvious that we can use it to operate servos. 

We can even replace one of the timing resistors with a potentiometer to control the duty cycle of the PWM output and get a full sweep of motion from the controlled servo. 

Simple, right? Well sadly, its not that straight forward :/


# The Problem with Astable Mode

