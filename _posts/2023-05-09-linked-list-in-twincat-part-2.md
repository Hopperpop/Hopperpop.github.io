---
layout: post
title: Linked list in TwinCAT - Part 2
date: 2023-05-09 12:54 +0000
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
In part 1, we talked about how to implement a general node that can be used as a link in a linked list. This part is going to be about the master node who controls the full list.
* Part 1 - Intro and node implementation
* Part 2 - Master node implementation
* Part 3 - Example usecase

## Master node
The master node is simply said the "point of contact" when someone wants to access the linked list. It will be responsible for keeping track of the list length and makes it simple to iterate over the list. The most important part is that the master node is always available, even if the list is empty. 

### - Master interface
The master node interface extends the normal node interface and adds two methods to it:
```iecst
INTERFACE ITF_LinkedList_Master EXTENDS ITF_LinkedList_Node
```
```iecst
METHOD mListAdd : BOOL
VAR_INPUT
	itfNode:	ITF_LinkedList_Node;
END_VAR
```
```iecst
METHOD mListRemove : BOOL
VAR_INPUT
	itfNode:	ITF_LinkedList_Node;
END_VAR
```
As the name implies they add and remove the given node from the list. Internally the list length will be updated, and the full link chain will be restored or extended. 

### - Master function block
The master node stores a link to the first and last element of the list. In the interface the mapped properties are still called `p_ListNext` and `p_ListPrevious` as with the normal node. But internally we call them `itfHead` and `itfTail` to make them more distinguished.     
It also adds an iterator variable and a link to the iterated node. The linked list is 0 indexed and new nodes are always added to the end of the list. 

```iecst
{attribute 'hide_all_locals'}
FUNCTION_BLOCK FB_LinkedList_Master IMPLEMENTS ITF_LinkedList_Master

VAR_OUTPUT
	ListLength:		UINT;	//Length of the linked list
END_VAR

VAR
	itfHead:		ITF_LinkedList_Node;
	itfTail:		ITF_LinkedList_Node;
	
	{attribute 'no_copy'}
	it:				UINT;					//Local iterator
	{attribute 'no_copy'}
	itfIt:			ITF_LinkedList_Node;	//Iterated node
END_VAR
```

```iecst
PROPERTY p_ListNext : ITF_LinkedList_Node
//----Get----
p_ListNext := itfHead;
//----Set----
itfHead := p_ListNext;
```
```iecst
PROPERTY p_ListPrevious : ITF_LinkedList_Node
//----Get----
p_ListPrevious := itfTail;
//----Set----
itfTail := p_ListPrevious;
```
Similar to the normal node we have a constant property `p_IsLinkMaster`, but this time it returns `TRUE`.
```iecst
PROPERTY p_IsLinkMaster : BOOL
//----Get----
p_IsLinkMaster := TRUE;
```
### - mListAdd

The `mListAdd` method will first check if the given node interface is valid. If the list is empty it will link both the tail and head from the master to the node, and the node's `p_ListPrevious` and `p_ListNext` to the master. They are literally pointing at each other. If the list already has elements, the last node gets linked to the new node. The new node links to the master. And also for the opposite direction: master tail -> new node -> previous tail node. After all of this the list length is incremented.
```iecst
METHOD mListAdd : BOOL
VAR_INPUT
	itfNode:	ITF_LinkedList_Node;
END_VAR
```
```iecst
IF itfNode <> 0 THEN
	IF ListLength = 0 THEN
		//--Empty list--
		itfHead 				:= itfNode;
		itfTail 				:= itfNode;
		itfNode.p_ListPrevious	:= THIS^;
		itfNode.p_ListNext		:= THIS^;
	ELSE
		//--Existing list--
		itfTail.p_ListNext 		:= itfNode;
		itfNode.p_ListNext		:= THIS^;
		itfNode.p_ListPrevious	:= itfTail;
		itfTail					:= itfNode;
	END_IF
	
	ListLength 					:= ListLength + 1;
END_IF
```

### - mListRemove

The `mListRemove` method checks also for valid interfaces, and just glues the previous and next node to each other. If the glueing is successfully it decrements the list size. Note that currently it doesn't checks if the given node is part of list. For example you can call this method on a master and give it a node from another list. The node will be removed from it's own list, but the length values won't be correct anymore for both lists. Normally the methods should only be called from within a node itself and be [`Internal`](https://infosys.beckhoff.com/content/1033/tc3_plc_intro/2530307467.html ) scoped, but as we are using interfaces we are limited to public methods. It wont cause any issues as long as it's not externally called.

```iecst
METHOD mListRemove : BOOL
VAR_INPUT
	itfNode:	ITF_LinkedList_Node;
END_VAR
```
```iecst
//Improve: Check if node is indeed in the list of this master

IF itfNode <> 0 THEN
	
	IF itfNode.p_ListPrevious <> 0 THEN
		
		IF itfNode.p_ListNext <> 0 THEN
			itfNode.p_ListPrevious.p_ListNext := itfNode.p_ListNext;
			itfNode.p_ListNext.p_ListPrevious := itfNode.p_ListPrevious;
		ELSE
			itfNode.p_ListPrevious.p_ListNext := 0;
		END_IF
		
		mListRemove := TRUE;
	END_IF

	itfNode.p_ListNext 		:= 0;
	itfNode.p_ListPrevious	:= 0;
END_IF

IF mListRemove THEN
	ListLength := MAX(0,ListLength - 1);
	
	//Reset iterator to beginning we don't know where the nodes is removed and recounting is needed
	it		:= 0;
	itfIt	:= itfHead;
END_IF
```

As said before the list is zero indexed. To know the position of a node, you need to count from the master until you reach the node. When removing a node we have no idea where we are removing the node. The internal iterator could point to a node before or after the deleted node. That is why on successful removal we reset the iterator to the start to make sure we count correctly.

### - mListGetNode
Now lets take a look at the most important method of the master `mListGetNode`. By calling this method with a given index, it will step through the list and return the interface of the node on that index. The last accessed node gets stored internally. When a second call happens, we now start from this node instead of the beginning. This will make iterating over the list with a `FOR` loop faster. To prevent accumulation of a index error (if they every happen), we always jump directly to the beginning when an index of 0 is requested. This also gives us a speed boost with restarting normal sequential iterations.

```iecst
METHOD mListGetNode : ITF_LinkedList_Node
VAR_INPUT
	i:		UINT; //0 => Length - 1
END_VAR
```
```iecst
//Reread/init
IF it = 0 THEN
	itfIt  	:= itfHead;
END_IF

//Reset to head
IF i = 0 THEN
	itfIt 	:= itfHead;
	it		:= 0;	
END_IF

WHILE it < i AND_THEN itfIt <> 0 AND_THEN NOT itfIt.p_IsLinkMaster  DO
	itfIt	:= itfIt.p_ListNext;
	it		:= it + 1; 
END_WHILE

WHILE it > i AND_THEN itfIt <> 0 AND_THEN NOT itfIt.p_IsLinkMaster DO
	itfIt	:= itfIt.p_ListPrevious;
	it		:= it - 1;
END_WHILE

IF itfIt <> 0 AND_THEN NOT itfIt.p_IsLinkMaster THEN
	//Node found, return interface
	mListGetNode := itfIt;
ELSE
	//Something went wrong, reset to head
	itfIt 	:= itfHead;
	it		:= 0;
END_IF
```
And with that we have the full code for our master.    

---
## First program
We can create a simple program to test out our FBs and see if they work in all occasions. By commenting in/out some nodes and making an online change, you can see that the list length variable changes correctly. We also double check this value by going over the full list. If the linking is broken, or some nodes aren't added the value of `i` wont match the list size.

```iecst
PROGRAM MAIN
VAR
	fbMaster:	FB_LinkedList_Master;
	
	fbNode_1:	FB_LinkedList_Node 	:= ( p_LinkMaster := fbMaster);
	fbNode_2:	FB_LinkedList_Node	:= ( p_LinkMaster := fbMaster);
	fbNode_3:	FB_LinkedList_Node	:= ( p_LinkMaster := fbMaster);
	//fbNode_4:	FB_LinkedList_Node	:= ( p_LinkMaster := fbMaster);

    i:			UINT;
	itfNode:	ITF_LinkedList_Node;
END_VAR
```
```iecst
i := 0;
WHILE (itfNode := fbMaster.mListGetNode(i)) <> 0 AND_THEN NOT itfNode.p_IsLinkMaster DO
	i := i + 1;
END_WHILE
```
Lets do the same thing again, but this time with an array of nodes. I like to refer to another of my [posts]({% post_url 2023-05-05-Init array in ST%}), where I give an overview on how to initialize arrays. 
```iecst
PROGRAM PRG_BareList
VAR CONSTANT
	LISTSIZE:	INT := 4;
END_VAR
VAR
	fbMaster:	FB_LinkedList_Master;
	fbNodes:	ARRAY [0..LISTSIZE-1] OF FB_LinkedList_Node := [LISTSIZE(( p_LinkMaster := fbMaster))];
...
```
Try to change the size now with an online change to ex. 5. You will see that master still reports a list size of 4 and similar for `i`. The new node in the array isn't added to the list. Do the same by decreasing the size to 3, and again nothing changes. **This is dangerous!** Because there is still a node in the list that still exist in memory (for now) but doesn't have a variable handle anymore. It could be that the same array is reused but just resized, or that a new array is created and we still pointing to the last element of the old array. Somehow we are still pointing to a memory location that could be overwritten anytime to contain something fully differently. We can expect to to trigger an exception anytime. 
> Don't use this linked link implementation with arrays, or at least don't resize the arrays during online change!
{: .prompt-warning }
**Solution**
: At the moment no. But it also doesn't make sense to use a linked list with arrays as we can simply use the array itself.    
    
**Cause**
: When resizing an array, the `fb_init` and `fb_exit` method of all the function blocks are called  with a `bInCopyCode = TRUE`. CodeSys/TwinCAT doesn't differentiate between new elements or removed elements. If the array is copied, all the elements are flagged as copy. A behavior that isn't fully correct and should be addressed in newer versions.    


## End
Now that we have the basis for a linked list. We still don't know how to extend this functionality to make it reusable in all our applications. 
In part 3 we will *lock all cars in our garage* and compare the performance of a linked list versus a standard array.
