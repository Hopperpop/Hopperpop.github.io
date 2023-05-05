---
layout: post
title: Init array in ST
date: 2023-05-05 12:05 +0200
categories: [Automation, IEC 61131-3 ]
tags: [IEC 61131-3, ST, Array, FB_init]
image: /assets/img/InitArray.png
---
---

In this post, I would like to go over the different ways to initialize an array of structures and function blocks. The main reason is that I found a syntax that wasn't well documented or, at least, not to my knowledge.

## Standard datatypes arrays
### - Basic example
Let's start simple by initializing an array of a standard data type.:
```iecst
VAR
	aIntArray:	ARRAY[0..4] OF INT := [0,1,2,3,4];
END_VAR
```
### - Partial initialization
If you only want to initialize the first few elements, you can do so without any issues by using the following syntax:
```iecst
VAR
	aIntArray:	ARRAY[0..4] OF INT := [0,1];
END_VAR
```
### - Repeated values
But what if you have many elements? Well, there is a shortcut that can be taken to define multiple elements with the same value. The syntax `X(.)` will repeat everything inside the brackets `X` times. Both of the following initializations create an array with similar values:
```iecst
VAR
	aIntArray_1:	ARRAY[0..9] OF INT := [5(0), 1, 2, 3(-1)];
	aIntArray_2:	ARRAY[0..9] OF INT := [0,0,0,0,0,1,2,-1,-1,-1];
END_VAR
```
## Multi dimensions
### - Multi-dimensional array
For arrays with multiple dimensions, you need to flatten the array before assigning:

```iecst
VAR
	aIntArray_1:	ARRAY[0..2, 0..1] OF INT := [ 0,1,2,3,4,5 ];
	(*	[
			[ 0 , 1 ],
			[ 2 , 3 ],
			[ 4 , 5 ]
		] 	*)
	IntArray_2:	ARRAY[0..2, 0..1] OF INT := [ 6(123) ];
	(*	[
			[ 123 , 123 ],
			[ 123 , 123 ],
			[ 123 , 123 ]
		]	*)
	aIntArray_3:	ARRAY[0..2, 0..1] OF INT := [ 2(0), 2(1), 2(2) ];
	(*	[
			[ 0 , 0 ],
			[ 1 , 1 ],
			[ 2 , 2 ]
		]	*)
	//Sadly the following syntax to have rows with the same value, doesn't work.
	aIntArray_4:	ARRAY[0..2, 0..1] OF INT := [ 3(0,1) ]; //Unexpected array initialisation
END_VAR
```
### - Array of array
Another option is to use ARRAY[.] OF ARRAY[.]. When accessing an element, you need to use `[X][Y]` instead of `[X,Y]`. During initial assignment, you can use `X([..])` to repeat rows, and `X([ Y(.) ])` to fill the whole array.

```iecst
VAR
	aIntArray_1:	ARRAY[0..2] OF ARRAY[0..1] OF INT := [ [0,1], [2,3], [4,5] ];
	(*	[
			[ 0 , 1 ],
			[ 2 , 3 ],
			[ 4 , 5 ],
		]	*)
	aIntArray_2:	ARRAY[0..2] OF ARRAY[0..1] OF INT := [ 3([ 2(123) ]) ];
	(*	[
			[ 123 , 123 ],
			[ 123 , 123 ],
			[ 123 , 123 ],
		]	*)
	aIntArray_3:	ARRAY[0..2] OF ARRAY[0..1] OF INT := [ 3( [0,1] ) ];
	(*	[
			[ 0 , 1 ],
			[ 0 , 1 ],
			[ 0 , 1 ]
		]	*)
	aIntArray_4:	ARRAY[0..2] OF ARRAY[0..1] OF INT := [ [2(0)], [2(1)], [2(2)] ];
	(*	[
			[ 0 , 0 ],
			[ 1 , 1 ],
			[ 2 , 2 ]
		]	*)
END_VAR
```

### - Constants
PLC programmers often use constants to define the size of their arrays. This way, they can easily update the size and reuse it in `FOR` loops. These constants can also be used to fill the array during initialization. The following examples show how to fill multi-dimensional arrays with 1.   
<ins>**NOTE**</ins>: When testing, line 10 resulted in a syntax error for `aIntArray_2`! The `X` needs to be surrounded by brackets, while the brackets are optional for `Y`. The logic behind this syntax can be confusing and may appear to be a bug.
```iecst
VAR CONSTANT
	X:		INT := 3;
	Y: 		INT := 2;
END_VAR

VAR
	//Works
	aIntArray_1:	ARRAY[1..X, 1..Y] OF INT		:= [ (X*Y)(1) ];
	//Doesn't works!!!!!
	aIntArray_2:	ARRAY[1..X] OF ARRAY[1..Y] OF INT 	:= [  X(  [  Y (1) ] )];
	//Works
	aIntArray_3:	ARRAY[1..X] OF ARRAY[1..Y] OF INT 	:= [ (X)( [  Y (1) ] )];
	//Works
	aIntArray_4:	ARRAY[1..X] OF ARRAY[1..Y] OF INT 	:= [ (X)( [ (Y)(1) ] )];	

	//Also works
	aIntArray_5:	ARRAY[1..(2*X)] OF INT 			:= [ (X-Y)(1), X(1), Y(1) ];
END_VAR
```

## Array of structures
### - Basic example
Let's define a simple structure:
```iecst
TYPE ST_Struct :
STRUCT
	xBool:		BOOL;
	lrReal:		LREAL;
END_STRUCT
END_TYPE
```    

For initialization we can do the following. Not all elements needs to be defined. The missing elements will stay at their default value.
```iecst
VAR
	aStruct: ARRAY [0..3] OF ST_Struct := 	[
							(xBool := TRUE),
							(lrReal := 1.0),
							(xBool := FALSE, lrReal := -3.0)
						];
END_VAR
```

### - Repeated elements
Now we are coming to the point where I (re-)discoverd the syntax for initializing an array of structures with repeated elements. This was the main reason why I created this blog post, as I haven't seen anyone else documenting it.
```iecst
VAR
	aStruct: ARRAY [0..5] OF ST_Struct := 	[ 
							4((xBool := TRUE)),
							(xBool := TRUE, lrReal := 5.0)
						];
END_VAR
```
Did you notice the double brackets? `X(.)` will repeat everything inside the brackets X times. As the structure also needs brackets, you end up with two brackets. It may seem obvious once you know it, but without any examples, most people (like me) will initially try it with single brackets, encounter a syntax error, and give up.

## Array of function blocks
Function blocks act in the same way as structures. You can initialize inputs, outputs, local variables, and even properties! Inputs and properties are a logical use, but outputs and local function block variables cannot typically be assigned during code execution. However, it is possible to do so during initialization. Personally, I consider it bad practice.
```iecst
FUNCTION_BLOCK FB_Test
VAR_INPUT
	iIn:	INT;
END_VAR
VAR_OUTPUT
	iOut:	INT;
END_VAR
VAR
	iVar:	INT;
	iProp:	INT;
END_VAR
```
```iecst
PROPERTY p_Prop : INT
//---Getter---
p_Prop 	:= iProp;
//---Setter---
iProp 	:= p_Prop;
```
To initialize an array of these function blocks:
```iecst
VAR										
	aFb_Test: ARRAY [0..1] OF FB_Test := [ 2((p_Prop := 5, iVar := 2, iIn := 6, iOut := 9)) ];
END_VAR
```
### - FB_init
Another way to initialize function blocks is to use the `FB_init` method. This method is called before the initial assignments and can be extended with custom parameters. The parameters can be mapped to internal variables for initialization. The key difference is that with FB_init, all parameters are required and can't be left as default. Depending on the use case, this can be beneficial or unwanted.  
    
Let's extend previous function block with the following `FB_init` method:
```iecst
METHOD FB_init : BOOL
VAR_INPUT
	bInitRetains: 	BOOL; // if TRUE, the retain variables are initialized (warm start / cold start)
	bInCopyCode: 	BOOL;  // if TRUE, the instance afterwards gets moved into the copy code (online change)
	iPara:		INT;
END_VAR
//------------------------------------------------
p_Prop := iPara;
```

These examples will call `FB_init` with the parameter `iPara`. This will set the `p_Prop` property, which in turn sets the internal variable `iProp`:
```iecst
VAR										
	//All these examples will both have p_Prop and iProp equal to 2
	aFb_Test1: ARRAY [0..1] OF FB_Test(2);
	aFb_Test2: ARRAY [0..1] OF FB_Test(iPara := 2);
	//Somehow this is also seen as 'init all fb's with the same value'
	aFb_Test3: ARRAY [0..1] OF FB_Test[(iPara := 2)]; 
	
	//Initialize the function blocks with different values
	aFb_Test4: ARRAY [0..1] OF FB_Test[ (iPara := 1),(iPara := 2) ];
	
	//Syntax error: FB-Init-Initializers(2) does not match the number of array-elements(3)
	aFb_Test5: ARRAY [0..2] OF FB_Test[ (iPara := 1),(iPara := 2) ];
	//Syntax error: Repeating of values can't be used with FB_init :-(
	aFb_Test6: ARRAY [0..2] OF FB_Test[ 3((iPara := 1)) ];
	
	//Both FB_init and property intizalization
	aFb_Test7: ARRAY [0..1] OF FB_Test(-1) := [ 2((p_Prop := 10)) ];
	//=> The fb's will have p_Prop and iProp equal to 10. 
	//Because FB_init is called first, and after that the initial assignments
END_VAR
```

## Conclusion
There are many ways to initialize arrays of standard and complex datatypes. Although it's possible to repeat the same value multiple times without the need to repeat it in code, the syntax for doing so is not well documented or consistent. It would be great if the language were updated to allow the failing examples to work.

If you know of or have found other examples that aren't listed here, please leave them in the comments below, and I will update this post accordingly.

Thank you for reading! If you found this post useful, please consider giving it a like.