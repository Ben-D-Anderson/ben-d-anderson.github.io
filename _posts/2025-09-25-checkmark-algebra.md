---
layout: post
title: Algebraic Formulae For The Perfect Checkmark Symbol
description: Providing simple formulas that can be used to calculate the coordinates for each point in a checkmark symbol, given an arbitrary bounding box.
readtime: 1 minute
toc: false
tags: maths, graphics, programming
---

In the case that you are required to render a custom checkmark for use in something like a checkbox, the formulas provided below will create a perfect result:

<img src="/assets/img/annoted-checkmark-coordinates.png"/>

To fill an area where the bottom-left corner is (0, 0):
- `THICKNESS` = `HEIGHT / 5`

- **A** - inner left
  - x = `THICKNESS * cos(45)`
  - y = `HEIGHT / 2`
- **B** - outer left
  - x = `0`
  - y = `HEIGHT / 2 - THICKNESS * sin(45)`
- **C** - outer elbow
  - x = `HEIGHT / 2 - THICKNESS * sin(45)`
  - y = `0`
- **D** - outer right
  - x = `WIDTH`
  - y = `WIDTH + THICKNESS * sin(45) - HEIGHT / 3`
- **E** - inner right
  - x = `WIDTH - THICKNESS * sqrt(2) / 2`
  - y = `WIDTH - HEIGHT / 3 + THICKNESS * sin(45) + THICKNESS * sqrt(2) / 2`
- **F** - inner elbow
  - x = `HEIGHT / 2 - THICKNESS * sin(45)`
  - y = `THICKNESS * sqrt(2)`

It's also worth noting that if you are planning to render the above checkmark using a graphics library then you may need to prepend `HEIGHT - (...)` to all the Y-values as graphics libraries typically consider the top of the screen to be Y=0 and the bottom to be Y=HEIGHT.
