# Factorio Cross-Mod Interop Spec

A specification for Factorio 2.0 mods to share item/fluid selection events and display recent selections from other mods.

## Overview

When a player selects an item or fluid in one mod's GUI, that mod broadcasts an event. Other mods that have subscribed to the event can track the selection and display it in a "recent items" panel for quick access.

## Implementing Mods

- [Bottleneck Analyzer](https://github.com/nickelbob/factorio-bottleneck-analyzer)
- [Project Cybersyn](https://github.com/nickelbob/project-cybersyn) (branch: `cross-mod-interop-recent-items`)
- [Recipe Book](https://codeberg.org/raiguard/RecipeBook) (fork with interop)

## Spec

See [spec.md](spec.md) for the full specification.
