---
title: "Try Again"
date: 2023-01-30T21:42:57+02:00
draft: false
featured: true
tags: ["C-Sharp", "Unity", "Game Programming"]
description: "Working on camera system in Unity with Cinemachine"
cover: "https://cdn.seppedekeyser.be/img/proj_GW-Overlord/Screenshot_3.png"
weight: 4
---
Try Again is a team game project at USC developed by 8 engineers (including me!) and several other art, marketting, legal, desgin students. 

It is 2.5D fast paced platformer where you play as Benny, a guy stuck in an unfinished game.

{{< youtube KzFYLFumL9Y>}}

## My Contribution

I assisted the lead engineer with Camera System and Object Pooling for levels.

### Camera System using Unity Cinemachine

When playing through the game majority of the camera movement is done with the help of Unity's Cinemachine. I helped setup the Cinemachine component in the scene making sure the virtual cameras switch smoothly.

At several places in the game the platformer style camera is switched to third person camera. I wrote a script to make this trasition smoother. I also wrote a script that overrides Cinemachine code to allow the player to rotate the virtual camera on key press. 

# 

```C#
 protected override void PostPipelineStageCallback(
        CinemachineVirtualCameraBase vcam,
        CinemachineCore.Stage stage, ref CameraState state, float deltaTime)
    {
        if (stage == CinemachineCore.Stage.Aim) // runs every frame
        {   
            var pos = state.RawOrientation;
            // you can modify camera rotation by changing pos values
            pos = Quaternion.Euler(m_XDefaultRotation, m_YDefaultRotation, m_ZDefaultRotation);
            state.RawOrientation = pos;
        }
    }
```

### Object Pooling

I set up a standard object pool in the Unity scene using C# to recycle the obstacle game objects in the level. 

## Release on Steam

Try Again is now live on Steam with overwheleminly positive reviews.

[TRY AGAIN on Steam](https://store.steampowered.com/app/2448340/TRY_AGAIN/)

Majority of the credit goes to the Try Again team. 

My contribution to try again was miniscule in comparison to the countless hours the other members of the team put in. 
