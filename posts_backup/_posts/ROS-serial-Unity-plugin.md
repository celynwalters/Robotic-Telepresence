---
title: 'ROS serial: Unity plugin'
date: 2016-11-21 21:04:48
tags:
---
Unity managed plugins.
Create 'Plugins' folder in Unity assets, and put `.dll` files in there.
Try to compile `rosserial_hello_world` as a `.dll`?

(Be careful with name mangling!?)

Create new empty project, Win32 DLL.
Add new `.cpp` file, `main.cpp`.
Put some code in:
```cpp main.cpp (rosserial_hello_world)
#include <iostream>
#define DLLEXPORT __declspec (dllexport)

extern "C" {
	DLLEXPORT int GetRandom() {
		return rand();
	}
}
```

Write a script in Unity and attach it to a `GameObject`.
In the script, call the function from the DLL:
```cs TestNative.cs
using UnityEngine;
using System.Collections;
using System.Runtime.InteropServices; //For DLLImport

public class TestNative : MonoBehaviour {
	[DllImport ("rosserial_unity")]
	private static extern int GetRandom();

	void Start () {
		print("Native random number: "+GetRandom());
	}
}
```
Run the scene.
I got a few errors, but the main one was:
```none Unity console
Failed to load 'Assets/Plugins/rosserial_unity.dll', expected 64 bit architecture (IMAGE_FILE_MACHINE_AMD64), but was IMAGE_FILE_MACHINE_I386. You must recompile your plugin for 64 bit architecture.
```
Simple, will recompile.
It worked!
![The output from the .dll in Unity](/Robotic-Telepresence/2016/11/21/ROS-serial-Unity-plugin/Random Number.png)

Now. Copy the `ros_lib` folder generated from `rosserial_windows` ([this post](/Robotic-Telepresence/2016/11/21/ROS-serial/)) into the Visual Studio project folder.
Add them to the project as before.
I also had to add `stdafx.cpp`, `stdafx.h` and `targetver.h` as they were missing. I'm not sure I need them, they were generated by Visual Studio for the console application.
The build succeeds with all the includes from the generated `rosserial_hello_world` code.

Splitting the original code into functions, and making the scope of created objects also allows compilation.
```cpp main.cpp (rosserial_unity)
#include "stdafx.h"
#include <string>
#include <stdio.h>
#include "ros.h"
#include <geometry_msgs/Twist.h>
#include <windows.h>
#define DLLEXPORT __declspec (dllexport)

using std::string;

extern "C" {
	ros::NodeHandle nh;
	geometry_msgs::Twist twist_msg;
	ros::Publisher cmd_vel_pub("cmd_vel", &twist_msg);

	DLLEXPORT void Connect() {
		char *ros_master = "192.168.0.31";
		printf("Connecting to server at %s\n", ros_master);
		nh.initNode(ros_master);
	}

	DLLEXPORT void Advertise() {
		printf("Advertising cmd_vel message\n");
		// geometry_msgs::Twist twist_msg;
		// ros::Publisher cmd_vel_pub("cmd_vel", &twist_msg);
		nh.advertise(cmd_vel_pub);
		printf("Go robot go!\n");
	}

	DLLEXPORT void Print() {
		twist_msg.linear.x = 5.1;
		twist_msg.linear.y = 0;
		twist_msg.linear.z = 0;
		twist_msg.angular.x = 0;
		twist_msg.angular.y = 0;
		twist_msg.angular.z = -1.8;
		cmd_vel_pub.publish(&twist_msg);

		nh.spinOnce();
	}
}
```
I made a Unity script, changed from the `TestNative.cs` script to accomodate the new functions and implement a while loop.
This should hopefully do near enough the same thing as the original `rosserial_hello_world` script.
```cs rosserial.cs
using UnityEngine;
using System.Collections;
using System.Runtime.InteropServices; //For DLLImport

public class rosserial : MonoBehaviour {
	[DllImport ("rosserial_unity")]
	private static extern void Connect();
	[DllImport ("rosserial_unity")]
	private static extern void Advertise();
	[DllImport ("rosserial_unity")]
	private static extern void Print();

	void Start () {
		Connect();
		Advertise();
		while(true) {
			Print();
			System.Threading.Thread.Sleep(1000); //This is bad but I'm just testing
		}
	}
}
```
This causes Unity to freeze, requiring a force quit with the Task Manager.
The play button is still depressed but not active, as if it's preparing to start.
HOWEVER, in the virtual machine, we get the output!
![Before the game was running, only up to the "Listening on port 11411" was visible. The output on the right is being streamed in.](/Robotic-Telepresence/2016/11/21/ROS-serial-Unity-plugin/Success.png)
Moving code to the `Update()` loop, and commenting out the functions still keeps causing Unity to hang.
The fix turned out to be simple.
We don't need a while loop in `Start()`, or in `Update()`. The `Update()` function is already a loop.
I set it to only `Print()` when the x component of the translation of Baxter's torso changed.
After connecting, the socket node complained every second about some desynchronisation:
```none Console output (excerpt)
[WARN] [1479767182.854363980]: Sync with device lost.
```
As soon as I moved baxter, it stopped.

Next step: Replace the dummy data with actual values!
