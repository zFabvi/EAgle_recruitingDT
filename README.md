# EAgle Digital Twin recruiting project: FS Vehicle Simulation

A Formula Student car simulation in **UE 4.26**: a track generated from a single spline, a vehicle with realistic physics, and an AI that drives it around the circuit with drivers implementation.

AI full track video [here](https://drive.google.com/file/d/1uHpTjMKPYyyaLhq5NPgnvL-OSedrS9UF/view?usp=drive_link).
User full trak video [here](https://drive.google.com/file/d/1mcBek7XS6nbp9PunedrY4b1y66H9QjQF/view?usp=drive_link).

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
Then, it casts 5 spheres in a fan in front of the car and picks the free direction closest to the track direction.

#### Following the track
At the start of play the controller caches the controlled vehicle, its movement component and the track spline. Every frame, `Calculate Steering` finds the closest point on the spline and looks ahead by a speed-dependent distance. It then computes the direction towards that look-ahead point and compares it with the car's forward vector. The angle between the two becomes a normalized steering value, scaled by a gain and clamped.
The look-ahead distance grows with speed, within a fixed range. At low speed the car reacts to a nearby point; at higher speed it looks further down the track, which keeps the steering calm and avoids constant small corrections.

#### Speed control

The controller maintains a target speed that is shared with the Event Graph. The target speed is reduced by how urgent the nearby obstacles are, by how hard the car is steering, and by how sharp the upcoming curve is.
The throttle and brake logic works in three zones. Below the target speed, the car applies a proportional amount of throttle instead of always using full throttle, so it approaches the target smoothly. Close to the target, it coasts with throttle and brake at zero. Above the target, it uses a small amount of negative throttle for gentle deceleration; if the overspeed is large, it uses the brake instead.
The negative throttle acts as engine braking and, at low speed, as reverse.

#### Corner anticipation

The controller samples the spline ahead of the car. It compares the tangent at the look-ahead point with the tangent a bit further down the track. The angle between those tangents is a simple measure of how sharp the upcoming curve is. The controller uses that angle to lower the target speed before the car reaches the corner.

This makes the speed profile much more natural: the car arrives at the corner already at a sensible speed, the steering does not have to fight the throttle, and the exit is cleaner. The effect is tuned with a preview distance, a maximum curve angle and a slowdown amount, so it can be adjusted per track without changing the logic.

#### Obstacle detection

The car looks ahead with a fan of five large sphere traces, spread over roughly ±45 degrees. The sphere shape was chosen because a thin line trace can pass above or beside a small cone; a sphere gives the trace some thickness, so it catches objects that are slightly off the ray.
Each hit is converted into a weight based on distance. Closer obstacles receive a higher weight. The maximum weight becomes the car's urgency. Urgency has two effects: it lowers the target speed, and above a threshold it marks the car as blocked. A blocked car receives a target speed of zero and can start the reverse recovery.

The hit directions are also summed into a lateral avoidance vector. That vector is added to the steering direction, so the car does not only slow down for an obstacle: it also steers around it. The combination of speed reduction and lateral correction is what makes the avoidance feel reasonably smooth rather than purely reactive.
At the current stopping point the five-sphere configuration is the final one. It works well at moderate speed, but at high speed a cone can occasionally be clipped if it appears late or falls between two spheres. A denser fan or a different trace shape would improve that, at the cost of more traces per frame. This is the main known trade-off of the current version.

#### Reverse recovery

When the car is blocked and its forward speed is almost zero, it reverses for a short, timed burst, then returns to normal driving. The reverse uses negative throttle and zero brake. The timer is important: without it the car could back up indefinitely. The burst is long enough to clear a cone or create space, but short enough to avoid driving blindly backwards for too long.

There is also a collision notification path. The vehicle can report a hit to the controller, which sets a temporary "hit" flag and allows the reverse recovery to start even if the obstacle was outside the front fan. This is useful for the cases where the car touches something with its side or after a turn, where the forward-looking traces no longer see the object. The path is prepared; its final timing and tuning are the natural next step.

#### Input model

The project's axis mappings define how the vehicle is meant to be driven: throttle on W/S with S producing a negative value, steering on A/D, and a separate brake input. The controller does not read those mappings directly; it writes to the vehicle movement component.

### 5. Sensors

Thanks to the User Widget feature in UE 4.26 I was able to map a Virtual Camera to the Viewport that costantly updates. To show some stats about the vehicle, some editable texts were added.
Throttle and Brake / HandBrake editable texts were buggy, so have been discarded from the implementation, waiting for a fix. Speed (in km/h) and RPM are correctly implemented.

## General Elements

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
| User Interface Widget                     | Show relevant stats about the vehicle|

## AI Use

I used DeepSeek V4.1 Flash 1017 as a support tool for a few bug fixes, for rewriting part of the project README documentation to make it clearer and more precise, and for refining the controller logic as the feature set grew more complex, while keeping the original base logic I had built intact. Some parts of the development still had to go through me because there were no dedicated or properly configured MCP tools for Linux, my development platform. The implementation, testing and final integration of the changes were done by me.

## Status

| Part                                         | Status                                         |
| -------------------------------------------- | ---------------------------------------------- |
| Track with variable width and closed circuit | Working                                        |
| Procedural kerbs in corners                  | Working, to be refined (tapered ends, margins) |
| Cones                                        | Working                                        |
| PhysX vehicle driven from the keyboard       | Working                                        |
| AI: spline following                         | Working                                        |
| AI: avoidance                                | Working, (the reverse recovery is still a bit buggy)|
| LIDAR and sensors                            | Working, improvable.                           |
