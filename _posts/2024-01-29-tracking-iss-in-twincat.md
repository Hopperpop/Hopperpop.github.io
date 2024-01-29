---
layout: post
title: Tracking ISS in TwinCAT
categories:
- Automation
- TwinCAT C++
tags:
- C++
- TwinCAT
- ISS
- SGP4
- HMI
image: "/assets/img/Sgp4/IssTracking.JPG"
math: true
date: 2024-01-29 23:05 +0000
---
To get familiar with writing C++ code in TwinCAT, I'm going to port a satellite tracking algorithm that can calculate the position of the international space station to TwinCAT, and let it run on the real-time of a PLC.

## Intro
During my last student years there were two things that caught my interest that would make a great combo:

ESP8266
: A cheap Wi-Fi capable module that can be programmed in a similar way as an Arduino.

Practical Engineering
: A youtuber called Practical Engineering (a.k.a Grady Hillhouse) who made a video explaining how he build a ISS pointer device. Read more about it on his [blog](https://practical.engineering/blog/2016/1/20/international-space-station-orbit-tracker-and-pointer). 

As Grady explained later in his blog, his design had two problems. First, it didn't had a source for the time. You can't calculate where a satellite is when you don't know what time it is. Secondly the algorithm uses initial orbital parameters to predict future positions. The North American Aerospace Defense Command (NORAD) job is to measure the position of all satellites and constantly update new orbital parameters as satellites drift over time from their theoretical orbit. This data is available online on sites like [Celestrak](https://celestrak.org/) in the form of a two-line-element ([TLE](https://wikipedia.org/wiki/Two-line_element_set)). An example of this can be seen below.

```plaintext
ISS (ZARYA)
1 25544U 98067A   24004.37386512  .00152947  00000+0  26263-2 0  9993
2 25544  51.6444  51.9798 0003888 345.7385 123.7761 15.50101737432991
```

The SGP4 (Simplified General Perturbations) algorithm that calculates the satellite position from a TLE has a drift of 1-3km a day[^SGP4]. For this reason it's important to use the latest TLE to prevent that the position is to far away from reality.

By using an ESP8266 those two problems can be easily solved. Connect to the internet over Wi-Fi, regular download the latest TLE and synchronize the time with a [NTP](https://wikipedia.org/wiki/Network_Time_Protocol) server. And that's all on a device that fits on your finger and only costs a few Euros. 

These all lead to building [Sattrack](https://hackaday.io/project/12607-iss-overpass-indicator) some years ago. A lamp that will glow up when the ISS is flying over. This was done by porting Grady's code to the new hardware. By following the [explanations](https://celestrak.org/columns/) of Dr. T.S. Kelso, some extra functionality was added that calculates the sun position to determine if the satellite was lit or hidden in the earth shadow, as well as an algorithm to predict past and future flyovers for a given location. All calculations were wrapped in a C++ class and added to a [library](https://github.com/Hopperpop/Sgp4-Library) for the ESP8266. The final touch was implementing a mini web server on the ESP8266 as an UI and connect some RGB led's for daily use. As this was done without prior C++ experience during my student time, I suggest not to have high expectation on the code quality.

Now let use this code as a fun project to explore the possibilities of TwinCAT C++.

---

## Implementation
### - Base project
We start by creating a standard C++ project with `Cyclic IO`. Follow the previous post [Hello world in TwinCAT C++]({% post_url 2024-01-07-hello-world-in-twincat-c %}) for a more detailed instruction. The new created module is named `TcSgp4`, after the original algorithm. 

In the `.tmc` file we define parameters, in- and output values. Except for the TLE values, they all have the type `LREAL`. Note that we can also give the module a custom 16x16 icon by clicking on the module itself in the module class tree.

![TcSgp4_tmc]({{ "/assets/img/Sgp4/TcSgp4_tmc.JPG" | relative_url }})

Run the TMC code generator to update all files in the project.

### - Source preparation
We add all source files from my previous [SGP4 library](https://github.com/Hopperpop/Sgp4-Library) to the project.

Sadly a copy/paste is not enough to get things working. We need to adapt the source in some way to make it compatible.

First we add in all new `.cpp` files the following two lines at the top[^TcPch].
```cpp
#include "TcPch.h"
#pragma hdrstop
```

Secondly, TwinCAT has it's own `math.h` functions. All standard mathematical functions need to be replaced by its TwinCAT counterpart. They can be recognized by the extra underscore.

```cpp
fabs()  => fabs_()
cos()   => cos_()
...
```

And last the `copysign()` function normally provided in `math.h` wasn't available and needs to be implemented. Online you can find source code for this function[^Gcc]. Just make sure you respect the license when copying code.

### - Code
In our `TcSgp4.h` file, you will find the module class definition. We start by including the Sgp4 library by adding its header at the top:
```cpp
#include "TcSgp4Interfaces.h"
#include "Sgp4.h"
```
At the bottom of it you will find the protected area of our module. Here we can define all internal variables. In this area you will already find autogenerated structures containing all module parameters, inputs and outputs. Create here an instance of the Sgp4 class called `cSat`.

```cpp
...
protected:
...
	///<AutoGeneratedContent id="Members">
	TcTraceLevel m_TraceLevelMax;
	TcSgp4SGP4 m_SGP4;
	TcSgp4Inputs m_Inputs;
	TcSgp4Outputs m_Outputs;
	ITcCyclicCallerInfoPtr m_spCyclicCaller;
	///</AutoGeneratedContent>

	// TODO: Custom variable
	Sgp4 cSat;
};
```

Now lets go to the code itself in the `TcSgp4.cpp` file. We first need to initialize our satellite tracking by applying the TLE and location parameters from our module.
```cpp
// State transition from PREOP to SAFEOP
//
// Initialize input parameters 
// Allocate memory
HRESULT CTcSgp4::SetObjStatePS(PTComInitDataHdr pInitData)
{
	m_Trace.Log(tlVerbose, FENTERA);
	HRESULT hr = S_OK;
	IMPLEMENT_ITCOMOBJECT_EVALUATE_INITDATA(pInitData);

	// TODO: Add initialization code
	cSat.site(m_SGP4.Site_Latitude, m_SGP4.Site_Longitude, m_SGP4.Site_Elevation);
	cSat.init("", m_SGP4.TLE1, m_SGP4.TLE2);

	m_Trace.Log(tlVerbose, FLEAVEA "hr=0x%08x", hr);
	return hr;
}
```

Next we update the `CycleUpdate` method to calculate the satellite position. With the `ipTask` interface we get access to the system time at the beginning of the task. This system time is in Windows `File time` format, a 64-bit value representing the elapsed time since 12:00 A.M. January 1, 1601 Coordinated Universal Time (UTC) in 100ns interval[^filetime]. TwinCAT system time is a copy of the local OS clock at startup, but runs independent from it. Changing the clock on the IPC won't change the system time.

Most astronomical calculations are done with `Julian date`, who counts the days since the start of the Julian period[^jd]. For that a conversion is needed between Julian date and file time. 

Additionally, let's add functionality to overwrite the time. The `Jd_In` input of the module can be set to zero, to use the system time, or used directly to be able to set an external time source.

Now with the time source available, the satellites position is calculated and written to the outputs variables.
```cpp
HRESULT CTcSgp4::CycleUpdate(ITcTask* ipTask, ITcUnknown* ipCaller, ULONG_PTR context)
{
	HRESULT hr = S_OK;
	signed long long _TaskSysTime;

	if (m_Inputs.Jd_In == 0) {
		// UTC at "begin of task" in 100 ns intervals since 1.1.1601
		hr = FAILED(hr) ? hr : ipTask->GetCurrentSysTime(&_TaskSysTime);

		//Convert filetime to julian date
		m_Outputs.Jd = ((_TaskSysTime - 116444736000000000LL) / 864000000000.0) + 2440587.5;
	}
	else {
		//Use input Julian date value
		m_Outputs.Jd = m_Inputs.Jd_In;
	}


	if (SUCCEEDED(hr)) {
		//Calculate satellite position
		cSat.findsat(m_Outputs.Jd);

		m_Outputs.Altitude	= cSat.satAlt;
		m_Outputs.Azimuth	= cSat.satAz;
		m_Outputs.Distance	= cSat.satDist;
		m_Outputs.Elevation = cSat.satEl;
		m_Outputs.Latitude	= cSat.satLat;
		m_Outputs.Longitude = cSat.satLon;
	}

	return hr;
}
```

An instance of the TcSgp4 module can be added to the project and initialized with the latest TLE.
![TcSgp4_Para]({{ "/assets/img/Sgp4/TcSgp4_Para.PNG" | relative_url }})

To make the outputs available over `ADS`, the `Create ADS Symbol` flag needs to be checked in the data area tab.
![Outputs_Ads]({{ "/assets/img/Sgp4/Outputs_Ads.PNG" | relative_url }})

## HMI
A simple visualization was build with Beckhoff's web HMI. A container was added to the main page with a world map as background. Inside the container a ellipse element was used to show the satellite position.

![Hmi_Struc]({{ "/assets/img/Sgp4/Hmi_Struc.JPG" | relative_url }})

The position is correctly marked on the map by mapping the latitude and longitude from our C++ module to the ellipse position.

![DotPos]({{ "/assets/img/Sgp4/DotPos.png" | relative_url }}){:.w-75}

Lastly a textbox element was mapped to display the formatted values.

![Hmi_ValueFormat]({{ "/assets/img/Sgp4/Hmi_ValueFormat.JPG" | relative_url }})


## Results

To check if the calculations are correct, I compared the results with a program called [Stellarium](https://stellarium.org/), an open source planetarium software. I made sure both sides used the same TLE and fixed the time on both sides. In a future post I will go in more details on how to automatically sync the clocks with an external server. As both programs give the same latitude and longitude, the algorithm seems to work.
![TwinCATvsStellarium]({{ "/assets/img/Sgp4/TwinCATvsStellarium.JPG" | relative_url }})


## References
[^SGP4]: [Simplified perturbations models](https://wikipedia.org/wiki/Simplified_perturbations_models)
[^TcPch]: [Using C++ classes in TwinCAT C++ module](https://infosys.beckhoff.com/content/1033/tc3_c/1053967499.html)
[^Gcc]: [Gcc mirror - copysign](https://github.com/gcc-mirror/gcc/blob/master/libiberty/copysign.c)
[^filetime]: [Windows File time](https://learn.microsoft.com/en-us/windows/win32/sysinfo/file-times)
[^jd]: [Julian day](https://en.wikipedia.org/wiki/Julian_day)