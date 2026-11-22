# EAgle Digital Twin recruiting project: FS Vehicle Simulation

A Formula Student car simulation in **UE 4.26**: a track generated from a single spline, a vehicle with realistic physics, and an AI that drives it around the circuit with drivers implementation.

## Goals

1. Build a reasonably realistic track from a single spline.
2. Simulate a Formula Student vehicle with believable dynamics, controllable via user code.
3. Implement an AI that follows the track and avoids obstacles, using sensors (raycasts/LIDAR, virtual camera), visualizing their data in realtime.

## Overview

The project is split into different parts:

### 1. Track

The track is a Blueprint with a spline. In the Construction Script, many road pieces (Spline Meshes) are created along the spline, so changing the layout only requires moving points in the editor. Each point has a scale, which allows widening the asphalt where needed. The track can be a closed circuit, controlled by a dedicated option.

### 2. Kerbs (EXTRA) and cones

Kerbs are generated procedurally . Along the spline, the change of direction from one point to the next is measured. Where the corner is tight enough, a kerb is placed on the inside of the bend, following the actual road width. The main parameter is the minimum corner radius.
A controller spawns cones along the track at random positions on the carriageway, using the spline as a reference. They have physics and can be knocked over.

### 3. Vehicle

The 3D model was downloaded and prepared in Blender (real-world scale, correct orientation, one root bone and one bone per wheel) and imported as a Skeletal Mesh. 
The rigging process was full of problems regarding orientation. To fix it (other than retrying multiple times) I found a useful [plugin](https://continuebreak.com/creations/ue-vehicle-rigging-addon-blender/) that helps with the entire process, from rigging to exporting (available in .fbx and .gltf).
In Unreal it uses the **PhysX Vehicles** system. The chassis is a single physics body, while the wheels are handled by Wheel Classes (radius, suspension, brakes, grip) and animated by an Animation Blueprint. Engine, gearbox, differential and steering are configured in the Vehicle Movement Component.

### 4. Artificial intelligence

Every frame it finds the closest point on the spline, picks a target further ahead (the distance grows with speed) and computes the signed angle between the car's heading and the target. That angle, normalized, becomes the steering command.
Then, t casts several rays in a fan in front of the car and picks the free direction closest to the track direction.

## Blueprint structure

| Element                                   | Purpose                              |
| ----------------------------------------- | ------------------------------------ |
| Track Blueprint                           | Spline, asphalt and kerbs            |
| Cone Blueprint                            | Cone with physics                    |
| Cone spawner Player Controller            | Spawns the cones                     |
| Game Mode                                 | Links controller and pawn            |
| Skeletal Mesh, Skeleton and Physics Asset | Vehicle                              |
| Wheel Blueprints (front and rear)         | Wheel parameters                     |
| Animation Blueprint                       | Visual wheel movement                |
| Vehicle Blueprint                         | Pawn with input and Vehicle Movement |
| AI Controller                             | Steering and avoidance               |

## Status

| Part                                         | Status                                         |
| -------------------------------------------- | ---------------------------------------------- |
| Track with variable width and closed circuit | Working                                        |
| Procedural kerbs in corners                  | Working, to be refined (tapered ends, margins) |
| Cones                                        | Working                                        |
| PhysX vehicle driven from the keyboard       | Working, tuning in progress                    |
| AI: spline following                         | Working                                        |
| AI: avoidance                                | Working                                        |
| LIDAR and sensors                            | WIP                                            |
