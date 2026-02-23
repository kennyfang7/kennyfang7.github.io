---
title: "Building a Configurable Game of Life in Python"
date: 2026-01-07
categories: [Projects, Finance]
tags: [python, pygame, cellular-automata, game-of-life]
description: "A configurable Conway's Game of Life simulator with live controls, editable rules, and a clean pygame interface."
---

I recently built a **Game of Life** project in Python to explore cellular automata in a way that is both visual and easy to tweak.

## What this project does

This implementation supports:

- **Configurable rules** (Conway, HighLife, Seeds)
- **Multiple neighborhoods** (Moore and von Neumann)
- **Boundary options** (wrap-around torus or dead edges)
- **Interactive pygame controls** for drawing, pausing, stepping, clearing, and randomizing

The core simulation logic is separated cleanly from rendering, so the same engine can be used with different UIs.

## Why I made it

I wanted a project that was simple in concept but rich in behavior. Game of Life is perfect for that: tiny local rules create surprisingly complex patterns.

It was also a good exercise in:

- writing modular Python code,
- building a small but usable UI, and
- designing a config-first workflow for experimentation.

## Stack

- Python
- pygame

## Running it

```bash
pip install -r requirements.txt
python runner.py
```

If you enjoy simulations, emergent behavior, or visual coding experiments, this was a fun build and a great foundation for future rule/pattern experiments.
