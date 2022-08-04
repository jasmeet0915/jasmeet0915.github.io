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

{{<figure src="/images/blogs/555_timer_duty_cycle/servo_tester.gif" loading="eager" width="400">}}

In this blog, I would like to share some interesting problems I faced and their solutions. But first, let's start with the basics.

# How Servo Motors Work 

So, a servo motor is basically a motor which can be moved to a particular angle. Internally, the servo uses a closed loop system to control the position of the shaft. Like any other closed loop system, this system also needs a feedback (position feedback in this case) to control the output. This is achieved with the help of a potentiometer. 

But how dow you send angles as commands to the servo? 

Well, a servo basically has 3 wires 2 of which are obviously VCC and GND for powering the internal DC motor. The third wire is where the magic happens. The third wire is the signal wire where you provide a PWM (Pulse Width Modulated) signal to get different angles. The relation between PWM signal and servo angle can be seen in the table below.

| Servo Angle (degrees) | Signal On Time (ms) | Period (ms) | Duty Cycle (%) |   |
|-----------------------|---------------------|-------------|----------------|---|
| 0                     | 1                   | 20          | 5              |   |
| 90                    | 1.5                 | 20          | 7.5            |   |
| 180                   | 2                   | 20          | 10             |   |

> **Note:** The above values are all ideal. In reality, I have even noticed the duty cycle for some servos to range from 2% to 14%!

So from this information we can conclude that to move a servo all we need is some power and a PWM signal. So how do we do that? We use the 555 Timer 😎 

> **Note:** We could also use a controller or a development board like arduino for this. Heck they even have in-built [libraries](https://www.arduino.cc/reference/en/libraries/servo/) so you don't even have to write the code that generates the PWM signal according to the above table. You just pass the angle to the `servo.write()` fucntion and sit back while the library does its magic. But, where's the fun in that :p

# 555 Timer and its Different Modes

