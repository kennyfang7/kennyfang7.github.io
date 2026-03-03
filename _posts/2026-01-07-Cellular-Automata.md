---
title: "Building a Configurable Game of Life in Python"
date: 2026-01-07
categories: [Projects, Programming, Simulations]
tags: [python, pygame, cellular automata, conway's game of life]
description: "A configurable Conway's Game of Life simulator with live controls, editable rules, and a clean pygame interface."
---

## What is Cellular Automata
Cellular automata are simple grid-based systems where each cell updates based on local rules and the state of its neighbors. Conway’s **Game of Life** is the classic example: there’s no “player” input once it’s running, but complex behavior emerges from a few deterministic update rules. I built this project to make those ideas tangible—something I could watch, pause, poke at, and reconfigure in real time.

I wanted to make it because it’s the kind of concept that’s easy to understand intellectually but far more interesting when you can experiment with it. Being able to draw patterns, swap neighborhoods, change boundary conditions, and flip between rule sets turns it from a textbook curiosity into a sandbox. The goal was a small, clean simulator that favors iteration: change a rule, hit step, see what happens.

Along the way I learned a lot about separating simulation from presentation, designing a UI that stays out of the way, and writing code that’s “configurable” without becoming messy. Cellular automata are unforgiving about performance and correctness—tiny bugs create wildly wrong outcomes—so the project pushed me to be disciplined with state updates, grid copying, and testing behavior with known patterns.

## Supplemental

**What the project supports**
- Configurable rules (Conway, HighLife, Seeds)
- Multiple neighborhoods (Moore and von Neumann)
- Boundary options (wrap-around torus or dead edges)
- Interactive pygame controls for drawing, pausing, stepping, clearing, and randomizing

**What I learned**
- Clean separation of concerns: simulation engine vs rendering/UI
- Correct update mechanics (avoiding in-place updates that corrupt the next state)
- How small rule changes can produce drastically different “macro” behavior
- Building a UI that optimizes for fast experimentation

**Stack**
- Python
- pygame

**Running it**

```bash
pip install -r requirements.txt
python runner.py
```

If you enjoy simulations, emergent behavior, or visual coding experiments, this was a fun build and a great foundation for future rule/pattern experiments.
