# Embedded AI HW5 Report

## Preface

Most of the professional FPV drone pilots fly in **Acro mode**. This mode works as follows:

* **Throttle** value determines the motors thrust. (Higher throttle means higher thrust)
* **Yaw** value rotates the drone in its plane. (Middle value means no change, lower value means constant left rotation, higher value means constant right rotation)
* **Roll** value rotates the drone around forward axis (the logic is the same as in **yaw**)
* **Pitch** value rotates the drone around the axis which is perpendicular to the forward axis and lies in the drone's plane. (the logic is the same as in **yaw**)

Let's focus on how roll and pitch stabilization works.

Each **Flight Controller** receives current roll and pitch absolute angles from a builtin IMU. Thus, it knows current absolute drone orientation.

When it receives **RC command** it adjusts absolute roll and pitch angles which it's going to maintain (for example, if the roll value is low, it lowers the absolute roll angle).

Then, it passes wanted angle and current angle (received from IMU) to the PID controller (with feedforward) which controls the motors. (Actually the system is more complicated, but generally it works like that).

That's how **Acro mode** works: by roll and pitch values you are able not to **set** the angle, but to **adjust** it.

## Problem

Imagine, there is an **External Control Unit** that sends **RC commands** to the **FC**, and it is working in **Acro mode**.
Thus, in order to maintain a specific roll and pitch angles, it needs current drone's orientation (from the IMU).
In general, FC sends mavlink with that data to the **Control Unit**.

However, every external connection is a serious vulnerability. The main reason is that any wired connection (including the connection between **FC** and **Control Unit**) can simply unsolder or be broken in any other possible way.

So, when the **Control Unit** loses the mavlink connection from the **FC**, it will no longer know absolute drone's position and therefore, no longer be able to control the drone in **Acro Mode** because, as I described below, by sending **Roll** and **Pitch** value you **adjust** the angle, not **set** the angle and you simply don't know in which way to adjust.

That's why I decided to do **Visual-based Auto-Leveling** which is able to stabilize drone if the mavlink connection is broken.

## Solution

### Step 1 - UDP video stream from Unity3D

#### Streaming using Unity3D API (the right way)

The most "right" way to stream video from Unity3D is to write a **custom plugin**. That means:
* Using Unity3D API to render each frame to the queue
* Use GStreamer to process each frame, encode it and send over UDP using RTP protocol

The **advantages** of this method are:
* Potentially low latency
* Scalable
* Low memory consumption

The **disadvantages** are:
* Hard to implement without previous Unity3D experience

And, unlucky for me, I have no experience with it. Thus, in order to skip a few weeks of development, I came up with another solution

#### Streaming using OBS (chosen)

The easiest way to stream is to use OBS studio with FFmpeg. The OBS studio will capture the frame from the Unity3D window and pass it to FFmpeg. In order to configure it I did the following steps:

* Go to **Settings/Output**
* Set **Output Mode** to `Advanced`
* Go to **Recording**
* Set **Type** to `Custom Output (FFmpeg)`
* Go to **FFmpeg Settings**
* Set **FFmpeg Output Type** to `Output to URL`
* Set **File path or URL** to `udp://{YOUR_ORANGE_PI_IP}:5012`
* Set **Container Format** to `mpegts`
* Set **Video Bitrate** to `2000 Kbps`
* Set **Keyframe interval (frames)** to `250`
* Set **Show all codecs (even if potentially incompatible)**
* Set **Video Encoder** to `h264_nvenc (libx264)`
* Set **Video Encoder Settings (if any)** to `preset=fast profile=high pix_fmt=yuv420p b=2000K rc=vbr zerolatency`

To start the stream just press `Start Recording`

And finally, you can view the video using the following pipeline:
```shell
 gst-launch-1.0 udpsrc port=5012 caps="video/mpegts, systemstream=(boolean)true, packetsize=(int)188" ! tsdemux ! h264parse ! avdec_h264 ! videoconvert ! autovideosink sync=false
```

The major disadvantage of this method is the latency: around **200ms** which is critical in realtime control systems. But, taking into account that I'm doing it only for studying purposes, it's sufficient in our case.

### Step 2 - Image processing on OrangePi

For horizon detection https://github.com/sallamander/horizon-detection is used.

This code processes the frame and returns the horizon line. 

Then, roll angle is the horizon rotation angle, and pitch angle can be taken as the middle point y-position transformed to angle using vertical camera FOV.

That gives us current roll and pitch angles.

### Step 3 - Control system (Auto-Level)

To auto-level the drone it is necessary to control both `raw` (left-right) and `pitch` (forward-backward).

The wanted absolute roll and pitch angles are zero, because we want to stabilize the drone.

Current roll and pitch angles is taken from the previous step.

Thus, having both wanted and current angles, we can use two simple P-controllers (PID controllers with zero I and D terms) to control the drone.

### Step 4 - Sending commands from OrangePi to control drone in Unity3D

Usually, in realtime control systems UDP is used, because you want to minimize the latency at all costs.

That's why OrangePi sends commands via UDP. Each command consists of two values:
* **Roll** value (uint16, min 174, max 1811)
* **Pitch** value (uint16, min 174, max 1811)

These min and max values were chosen, because it's what most of **RC transmitters** send.

### Step 5 - Receiving commands in Unity3D

In order to simplify the task, drone has fixed altitude and only roll/pitch adjustment is received from UDP.

### Results

In general, OrangePi is able to process UDP video stream in realtime with **30FPS** and **auto-level the drone successfully**.

However, due to high latency, the P coefficient for P-controller is chosen relatively low. Otherwise, the drone will oscillate and become uncontrolled.

Also, sometimes, the horizon is detected badly or even not detected at all. That's why, in order to finalize this project, **horizon detection must be improved**. Also, it needs to be tested in different time and weather conditions.

Nevertheless, this project showed good results with non-complicated landscapes and can be further improved for using in production.

Sample video:
[Video](assets/video.mp4)
