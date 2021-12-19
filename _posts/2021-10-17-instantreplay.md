---
layout: post
title: InstantReplay
description: Minecraft plugin to replay server events
tags: project
---

[GitHub](https://github.com/Ben-D-Anderson/InstantReplay)

InstantReplay is the largest project I have written to date, with lots of code, extensive documentation and a thorough wiki.

InstantReplay is a Minecraft plugin for Spigot versions 1.8 to 1.17 (inclusive) which allows Minecraft server administrators to review events by re-watching them.

This plugin is ideal for Minecraft servers running game-modes such as Factions and HCF, as players often break rules which would need recorded proof of the player violating the rule in order to punish them. Gone are the days of recording your screen to catch rule-breakers, server admins can now re-watch the rule violation occurring in-game.

InstantReplay is split into three different theoretical modules:
1. Loggers - observe events occurring in the world and record them to the event database.
2. Providers - when a replay is started, providers will fetch the required events from the event database, parse it and make any necessary adjustments (such as player movement predictions).
3. Renderers - render events acquired from a provider to the player running the replay using packets and multi-version implementations.

Features:
- View player's inventory (right click them)
- View rough player movements
- View block changes
- View player joining and leaving
- View skins
- Parsing to timestamp
- Getting current timestamp
- Change speed of replay
- Change radius of replay
- Choose time to play replay from (either from time unit ago or UNIX timestamp)
- Pause and resume replays
- Death and damage events in chat
- Optimized for performance
- Highly configurable