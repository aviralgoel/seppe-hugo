---
title: "Minigin Engine"
date: 2022-11-11T18:06:18+01:00
draft: false
featured: false
tags: ["C++", "SDL 2.0"]
description: "A simple 2d game engine, written from scratch in C++ using SDL."
cover: "https://cdn.seppedekeyser.be/img/proj_Minigin/BubbleBobble_game.png"
weight: 5
---

A simple 2d game engine, written from scratch in C++ using SDL. (GitHub project link: https://github.com/SeppahBaws/Minigin)



### Features set

- SceneManager (with the ability to easily switch between scenes)
- GameObject-Component system
- Animated sprites rendering
- Flexible input system
- Physics (utilizing [Box2D](https://box2d.org))
- Debug UI (utilizing [Dear ImGui](https://github.com/ocornut/imgui))
- Simple Settings Manager



![Spawning a simple bubble while the enemy is going through its AI loop](https://cdn.seppedekeyser.be/img/proj_Minigin/BubbleBobble_bubble.gif)



### Engine design

The engine utilizes a GameObject-Component system, with the game developer being able to write custom components.

### Component order of execution

These are the main functions exposed by BaseComponent. Custom scripts can override these to implement 
```
OnPrepare()

OnPhysicsUpdate()    <---+
OnUpdate()               |  Main Game Loop
OnRender()               |
OnImGui()            ----+

OnCleanup()

// Physics callbacks:
OnCollisionBegin(...)
OnCollisionEnd(...)
```



### Input system

I spent a lot of time on the input system. I wanted it to be both easy to set up, but also have support for both keyboard/mouse and a gamepad. It is an encapsulation around XInput and SDL's input system. Yes, SDL also has support for XBOX controllers but i found that it left much to be desired which is why I used XInput.

The input system has support for axis mappings (inputs that have a range, like one of the triggers which have inputs varying ), and action mappings (inputs that fire once, like a button).

Setting up input bindings is really easy:

```cpp
InputManager::GetInstance().SetupAxis("MovementHorizontal", {
    {
        { Key::D, 1.0f },
        { Key::A, -1.0f },
        { GamepadAxis::LeftThumbX, 1.0f },
    }
});

InputManager::GetInstance().SetupAction("Jump", {
    {
        { Key::Space, InputState::Pressed },
        { GamepadButton::A, InputState::Pressed },
    }
});
```

Axises get defined by either a Keyboard key or a Gamepad axis, and "scale". The scale is used to set up two buttons in one axis so that we can set up a horizontal movement like above.

An Action is defined by a Keyboard key, Mouse button, or a Gamepad button, and an input state. The InputState allows for actions to get called based on whether the button was pressed or released etc.

Then later on in the player behavior you can poll for 

```cpp
InputManager& input = InputManager::GetInstance();
if (input.GetAction("Jump"))
{
    // Jump
}

float horizontal = input.GetAxis("MovementHorizontal");
// Move the player horizontally.
```