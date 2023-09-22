---
title: "Try Again Game"
date: 2023-01-11T18:03:40+01:00
draft: false
featured: true
tags: ["C#", "Unity", "Game Programming"]
description: "Working on camera system in Unity with Cinemachine"
cover: "https://cdn.akamai.steamstatic.com/steam/apps/2448340/header.jpg"
weight: 6
---

Try Again is a team game project at USC developed by 8 engineers (including me!) and several other art, marketting, legal, desgin students. 

It is 2.5D fast paced platformer where you play as Benny, a guy stuck in an unfinished game.

{{< youtube KzFYLFumL9Y>}}

## My Contribution

I assisted the lead engineer with Camera System and Object Pooling for levels.

### Camera System using Unity Cinemachine

When playing through the game majority of the camera movement is done with the help of Unity's Cinemachine. I helped setup the Cinemachine component in the scene making sure the virtual cameras switch smoothly.

![](https://cdn.akamai.steamstatic.com/steam/apps/2448340/ss_b607aa2e8161c8dd57bda30a6488907af3e8bcf6.600x338.jpg)

At several places in the game the platformer style camera is switched to third person camera. I wrote a script to make this trasition smoother. I also wrote a script that overrides Cinemachine code to allow the player to rotate the virtual camera on key press. 



![](https://cdn.akamai.steamstatic.com/steam/apps/2448340/ss_b6cc256d49eb2b02eb14513a848be2dbfe269ef4.600x338.jpg)

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



![](https://cdn.akamai.steamstatic.com/steam/apps/2448340/extras/shipping-containers_gif_.gif)

## Release on Steam

Try Again is now live on Steam with overwheleminly positive reviews.

[TRY AGAIN on Steam](https://store.steampowered.com/app/2448340/TRY_AGAIN/)

Majority of the credit goes to the Try Again team. 

My contribution to try again was miniscule in comparison to the countless hours the other members of the team put in. 
