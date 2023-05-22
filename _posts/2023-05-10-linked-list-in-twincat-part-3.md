---
layout: post
title: Linked list in TwinCAT - Part 3
date: 2023-05-10 13:06 +0000
mermaid: true
categories:
- Automation
- IEC 61131-3
tags:
- IEC 61131-3
- ST
- Array
- OOP
- Linked list
---
In the last part of this blog series, we use our linked list in basic application and test the performance of the extra overhead.
* Part 1 - Intro and node implementation
* Part 2 - Master node implementation
* Part 3 - Example usecase


## Locking cars
Imagine we have a collection of cars. Some cars you own in your own garage. All the rest are part of a company's car fleet. Both you and the company likes to have the ability to lock and unlock all owned cars.


### - FB_Car
First we create an interface for our cars. It's not necessary, but it will improve type safety when multiple function block definitions (classes) are used. 
```iecst
INTERFACE ITF_Car
    PROPERTY p_Locked : BOOL
```
The car function blocks is very simple. It contains a local variable to store the lock state. Implements `ITF_Car` interface, where the `p_Locked` property is directly mapped to the internal state. To make online viewing easy we add the `{attribute 'monitoring' := 'call'}` pragma (see [InfoSys](https://infosys.beckhoff.com/content/1033/tc3_plc_intro/2529692299.html)). The function block also extends our previous created `FB_LinkedList_Node`, to make all cars a potential list node.
```iecst
{attribute 'hide_all_locals'}
FUNCTION_BLOCK FB_Car EXTENDS FB_LinkedList_Node IMPLEMENTS ITF_Car

VAR
	Locked:		BOOL;
END_VAR
```
```iecst
{attribute 'monitoring' := 'call'}
PROPERTY p_Locked : BOOL
//----Get----
p_Locked := Locked;
//----Set----
Locked := p_Locked;
```

### - FB_CarGroup
The car group function block will be our master node of the linked list. It extends `FB_LinkedList_Master` and also implements `ITF_Car`. When writing the lock property it will (un)lock all cars! Reading it will return if all cars are locked. It possible to give this property another name like "p_LockAll" but by using the same interface and property name you create an abstraction between a single car and a car group. Neat!
```iecst
FUNCTION_BLOCK FB_CarGroup EXTENDS FB_LinkedList_Master IMPLEMENTS ITF_Car
VAR
END_VAR
```
`p_Locked` get
```iecst
VAR
	i:		UINT;
	itfCar:	ITF_Car;
END_VAR
```
```iecst
p_Locked := TRUE;

FOR i := 0 TO ListLength DO
	IF __QUERYINTERFACE( mListGetNode(i), itfCar) THEN
		p_Locked R= NOT itfCar.p_Locked;
	END_IF
END_FOR
```
`p_Locked` set
```iecst
VAR
	i:		UINT;
	itfCar:	ITF_Car;
END_VAR
```
```iecst
FOR i := 0 TO ListLength DO
	IF __QUERYINTERFACE( mListGetNode(i), itfCar) THEN
		itfCar.p_Locked := p_Locked;
	END_IF
END_FOR
```
### - __QUERYPOINTER & __QUERYINTERFACE
The `__QUERYPOINTER` and `__QUERYINTERFACE` are both extensions of IEC61131-3. They aren't used a lot, but they provide some interesting OOP mechanisms. These operators make it possible to make a clear separation with the linked list implementation and the function block that extends the list.    

`__QUERYPOINTER` will return the pointer to the function block that implements the given interface. If we ask the master node to return the node interface of the first element. We can now convert that interface to a pointer to the containing function block. In our example a pointer to `FB_Car`. By having the pointer we have full control over the car. The return value of this operator is a boolean that let us know if the casting was successful.  

> For the operator to work, the requested interface need to extend `__System.IQueryInterface`. In part 1 we did this with the ITF_LinkedList_Node interface.
{: .prompt-tip }
    
The problem with using pointers is that they are not type safe. There is nothing that checks if the returned pointer is a car.  It only knows that the function block has implemented the ITF_LinkedList_Node interface. It possible that we expect a car, but it's actually a bicycle. And calling a car method on a bicycle, will cause an exception, putting your garage on fire!    
The solution: `__QUERYINTERFACE`. This operator does the same thing but goes one step further. It checks if the found function block implements another interface. Basically we ask if it can convert the node interface to a car interface. If it's a yes, we lock the car. If it's a no because the node is a bicycle, we do nothing. This gives us the power to mix types in the linked list. If this isn't OOP, than I don't know what OOP is.    

## Our first list
Now that everything is explained lets just build our garage with our favorite cars. And test the functionality to lock a single car, or all cars.
```iecst
PROGRAM MAIN
VAR
	fbGarage:		FB_CarGroup;
	fbCorvette:		FB_Car := ( p_LinkMaster := fbGarage );
	fbMustang:		FB_Car := ( p_LinkMaster := fbGarage );
END_VAR
```

![First garage]({{ "/assets/img/LinkedList/FirstGarage.webp" | relative_url }})

Instead of always mapping a car to a car group, lets create a default list `fbCarFleet` that all cars are automatically added to. It's still possible to move cars explicitly to our private garage. 
```iecst
PROGRAM MAIN
VAR
	fbCarFleet:		FB_CarGroup;
	fbCar_Red:		FB_Car;
	fbCar_Green:	FB_Car;
	fbCar_Blue:		FB_Car;
	
	fbGarage:		FB_CarGroup;
	fbCorvette:		FB_Car := ( p_LinkMaster := fbGarage );
	fbMustang:		FB_Car := ( p_LinkMaster := fbGarage );
END_VAR
```
To make this work, we add a simple `FB_init` to `FB_Car`. We don't need to think about online changes as the child linked list function block handles this on its own. When the master node is defined explicitly, `FB_init` will first add it to the default list. After the initial assignment the node is moved to the new list.
```iecst
METHOD FB_init : BOOL
VAR_INPUT
	bInitRetains : 	BOOL; // if TRUE, the retain variables are initialized (warm start / cold start)
	bInCopyCode : 	BOOL;  // if TRUE, the instance afterwards gets moved into the copy code (online change)
END_VAR

p_LinkMaster := MAIN.fbCarFleet;
```
The result:    
![First garage]({{ "/assets/img/LinkedList/CarFleet.webp" | relative_url }})

For the last example we will move nodes from one list to another. Lets start selling and buying cars from our private garage the the car fleet.
```iecst
PROGRAM MAIN
VAR
	fbCarFleet:		FB_CarGroup;
	fbCar_Red:		FB_Car;
	fbCar_Green:	FB_Car;
	fbCar_Blue:		FB_Car;

	fbGarage:		FB_CarGroup;
	fbCorvette:		FB_Car := ( p_LinkMaster := fbGarage );
	fbMustang:		FB_Car := ( p_LinkMaster := fbGarage );

	//Triggers
	xSellCar:		BOOL;
	xBuyCar:		BOOL;
END_VAR

VAR_TEMP
	{attribute 'hide'} 
	pCar:			POINTER TO FB_Car;
END_VAR
```
```iecst
IF xSellCar THEN
	xSellCar := FALSE; //Reset trigger
	//Get last car in the garage list and sell it to the fleet
	IF __QUERYPOINTER(fbGarage.mListGetNode( fbGarage.ListLength - 1), pCar) THEN
		pCar^.p_LinkMaster := fbCarFleet;
	END_IF
END_IF

IF xBuyCar THEN
	xBuyCar := FALSE; //Reset trigger
	//Get first car from the fleed and add it to the garage
	IF __QUERYPOINTER(fbCarFleet.mListGetNode(0), pCar) THEN
		pCar^.p_LinkMaster := fbGarage;
	END_IF
END_IF
```
![First garage]({{ "/assets/img/LinkedList/SellBuyCars.webp" | relative_url }})

## Performance
We can expect that the linked list with all the extra code will be slower than iterating over an array. To understand how much different, create a simple program that iterate over 10_000 cars.
```iecst
PROGRAM PERFORMANCE
VAR CONSTANT
	iSize:		INT := 10000;
END_VAR

VAR
	fbMaster:	FB_CarGroup;
	fbCars:		ARRAY[0..iSize-1] OF FB_Car := [ iSize(( p_LinkMaster := fbMaster )) ];
	i:			INT;
	itfCar:		ITF_Car;
	
	Profiler_List: 	PROFILER;
	Profiler_Array:	PROFILER;
END_VAR
```
```iecst
MAX_AVERAGE_MEASURES := 100;

//-----Linked list iteration----
Profiler_List( START := TRUE );
FOR i := 0 TO iSize - 1 DO
	IF __QUERYINTERFACE( fbMaster.mListGetNode( i ), itfCar) THEN
		itfCar.p_Locked := TRUE;
	END_IF
END_FOR
Profiler_List( START := FALSE );

//-----Array iteration----
Profiler_Array( START := TRUE );
FOR i := 0 TO iSize - 1 DO
	fbCars[i].p_Locked := TRUE;
END_FOR
Profiler_Array( START := FALSE );
```

The result are 50µs for array iteration and 427µs for list iteration. A factor of 8.5 times slower. As long as the list isn't to big, this won't cause a big issue on modern hardware.

## Closing word
Linked list bring some interesting possibilities to plc programming that before where hard to achieve because of the static memory allocation. The basic function blocks can be extended and reused in different ways. Some inspiration:
* Have the master node embedded and encapsulated in every node, by defining it as `VAR_STAT` variable. This way the list is protected from outsiders. I like to see this as a *hivemind* in a swarm.
* Mix node types in one list. Have a group of all actuators (servos, pneumatic valves, ...) in a machine station, and be able to reset them all.
* Make it possible to link a node to multiple lists. This way you can group all actuators in a station, but also group all servos in the whole machine together to act on them.
* Create a linked list of linked lists.
* Extend the codebase to have more control over the list order. 
* Create a function block for a [XPlanar](https://www.beckhoff.com/en-en/products/motion/xplanar-planar-motor-system/) or [XTS](https://www.beckhoff.com/en-en/products/motion/xts-linear-product-transport/) mover and link them to the station they are currently part of.
* ...

As final words I wanted to state that this work isn't peer reviewed and hasn't been tested in production. It's still possible that it contain bugs that can trigger an exception. Use with care, improve and re-share!
