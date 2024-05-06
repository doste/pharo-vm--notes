### How to create a new primitive and (micro-)benchmark it?

In this tutorial we'll see how to implement a new primitive. 
 To do that we'll implement a concrete real (even a useful! :P) primitive. We'll see all the steps we have to go through to not only implement it but also test it and (micro) benchmarking it. We'll also see all the components involved in the process and how they interact with each other.

 ##### primitiveFormat
 
The primitive we are about to implement gives us the format of the class of a given object.

###### The format of an object

All objects in Pharo have a 'Header', this gives us all kinds of information about the object. The VM is the component that is the most interested in this data. From a user point of view (from the image side for example) the header of an object is not even accessible.
 In this header, among other things, there is a field called the *format*: This value describes the layout in memory of an object. In a 64-bit VM, the header of an object has 64 bits. 5 of these 64 bits are used for the format.
 For this tutorial that is all you need to know about the format of an object, but if you are curious see:
 - [A presentation made by the PharoVM team about the layout of objects](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-09-29-VM-Hernandez-objectsLayout.pdf)
 - `Behaviour >> #instSpec` : The comment in this method describes each format.

 ##### Back to primitiveFormat:

 As you may already know, the PharoVM works not only with an interpreter but also with a JIT compiler. So, most of the implementation of methods is done twice: one for the intepreter and one the JIT compiler.

 The two 'units of work' of the VM are: bytecodes and primitives.

 When either of the interpreter or the JIT compiler is executing some Pharo code, they are dealing with bytecodes or with primitives.

 _We'll see how to implement our own new bytecodes in another tutorial._

Then, to implement a primitive it would make sense to implement it for the intepreter *and* for the JIT compiler. So, we'll need (at least) two methods:
`primitiveFormat` and `genPrimitiveFormat`. 
The naming is just a convention. 
If you navigate to the `InterpreterPrimitives` class you can see all the methods there corresponding to each primitive implemented for the interpreter.
To see all the primitives implemented by the JIT compiler you need to go to the `CogObjectRepresentation` class.
_The convention is for the jitted ones to have a `gen` at the beginning. This way we can distinguish them just by the name._


 ##### Implementing IntepreterPrimitives >> primitiveFormat

 Let's start with the interpreter one:

 As mentioned earlier if we want to access the format of an object we need to access to its header. Fortunaly there are a couple of helper methods for exactly that.
 The implementation of the primitive looks like this:
 ```
        IntepreterPrimitives >> primitiveFormat

	| receiver header format |
	receiver := self stackTop.
	(objectMemory isImmediate: receiver) ifTrue: [ ^ self primitiveFail ].
	header := objectMemory baseHeader: receiver.
	format := objectMemory formatOfHeader: header.
	self
		pop: argumentCount + 1
		thenPush: (objectMemory integerObjectOf: format)
```
When the primitive is executed we have the receiver in the top of the stack, this receiver is the one who "called" the primitive. We check that the receiver is not an immediate object (1). [[Explanation below]]
 Then we extract the header from it, and then from the header the format. Finally we pop the receiver and pushed the result, this is the format of the receiver. (argumentCount is 0 in this case, this primitive doesn't take any argument).

Pretty simple, right? Most of the work is done by the helper methods that give us the header and the format of the object.

Now, let's move to the jitted one.

##### Implementing CogObjectRepresentation >> genPrimitiveFormat

Now instead of writing code that will be executed when the intepreter "calls" the primitive, we need to write code that when the JIT compiler calls it, it will in turn generate machine code that then will be executed.
 This detail is *very* important when dealing with the JIT compiler: we are writing code that will write another code, this last code will be machine code.
 So when you see a message send like `cogit CallR: $someRegister` this will *not* execute the call, this would generate code *to* execute the call.

```
genPrimitiveFormat
	| jumpImmediate primitiveFailure |
	"Only works if the instance is not forwarded (it's been already resolved)'"

1	cogit methodNumArgs > 0 ifTrue: [ ^ UnimplementedPrimitive ].

	"Immediates don't have format"
2	jumpImmediate := self genJumpImmediate: ReceiverResultReg.

	"We obtain the header of the receiver, and then from the header we obtain the format:"
	"Get the entire word with the header in ReceiverResultReg"
3	cogit MoveMw: 0 r: ReceiverResultReg R: ReceiverResultReg.
	"Mask off format"
4	cogit
		LogicalShiftRightCq: objectMemory formatShift
		R: ReceiverResultReg.
5	cogit AndCq: objectMemory formatMask R: ReceiverResultReg.

	"Now in ReceiverResultReg we have the 5 bits representing the format of the instance.
	We have to convert that value to a Pharo Number Object (SmallInteger)"
6	cogit
		LogicalShiftLeftCq: objectMemory numSmallIntegerTagBits
		R: ReceiverResultReg.
	"To do that, we shift left 3 for the tag and then tag it as 001, which is the tag for SmallInteger"
7	cogit AddCq: 1 R: ReceiverResultReg.
	"In ReceiverResultReg we have the SmallInteger representing the format. We are done. "

8	cogit genPrimReturn.

	primitiveFailure := cogit Label.
9	jumpImmediate jmpTarget: primitiveFailure.

	^ CompletePrimitive
```
This one seems a little more tricky, but if you pay attention the logic is exactly the same as in `primitiveFormat`, the difference is the code we writo to express it.
Don't worry we'll break it down line by line:

Now, as we are in _JIT land_ we receive the receiver through a specific register: `ReceiverResultReg`.

[Keep in mind that all these registers are 'virtual', this means that at this stage they are not the actual processor registers, that's why the names]

First, in line 1 and 2 we do the same check as in `primitiveFormat`.

In line 3, with `cogit MoveMw: 0 r: ReceiverResultReg R: ReceiverResultReg.` what we are doing is accesing the memory pointed by the value stored in `ReceiverResultReg`
at offset 0 reading a word (64 bits in a 64-bit machine) and then storing whatever value is in there in the same register.
 If you understood that last sentence feel free to skip this section, but I'll dive into that line because the first time you see it, it may be a little confusing:
 
 ```
 Cogit >> #MoveMw: offset r: baseReg R: destReg
   This method will generate code for what is basically a 'load' from memory.
 It will access the memory pointed by the value in baseReg, adding offset to that address, and then it will read a word from that memory address. That word will then be stored in the destReg.
 How to know how much to read from the memory address? It's in the name of the method: here we have a 'w' after the MoveM (M for memory) so we kwow it will read an entire word.
 There is also MoveMb:r:R for example; this one will read just a byte from the given memory address. To see all the methods refer to the Cogit class.

 ```

So, back to the primitive, what we are doing with the line `cogit MoveMw: 0 r: ReceiverResultReg R: ReceiverResultReg.` is accessing to the memory pointed by the `ReceiverResultReg`. This register contains an object (_a pointer to_ an object) and we know every object has a header at the beginning, and we know that the size of that header is 64 bits for all objects (in a 64-bit machine). So with that line we are storing the actual header inside the `ReceiverResultReg`.

Then, in line 4 and 5 we access to the format of that header. To do so we do a little of bit-twiddling, nothing fancy: just ANDing with 1's to obtain the bits corresponding to the format.

```
               | x x x x x x ...  f1 f2 f3 f4 f5 x x x ... x x x |  <- header    (f1...f5 are the 5 bits corresponding to the format)
        AND    | 0 0 0 0 0 0 ...  1  1   1  1  1 0 0 0 ... 0 0 0 |
        -------------------------------------------------------------
               |                 f1 f2 f3 f4 f5 0 0 0  ... 0 0 0 |  <- result of applying the mask 

```
Then in line 6 and 7 we convert the result value to a Pharo integer (see (1) if this doesn't make sense for you).

If we arrive to line 8 then it's a success, and we should just return. The result value goes through the same register as we receive the receiver (that's why the name xD)

Line 9 is where we end up if we had any errors. This is how we communicate that the primitive has failed, by not executing `genPrimReturn`.


##### Installing the primitives

What we just did was to implement the primitives, but we need to tell to the intepreter and to the JIT compiler about these new primitives we made for them.
The way we do that is by putting them in a table.
 Each of them have a specific table for all the primitives. The method to access them has the same name: `initializePrimitiveTable`.
As you can imagine in `Cogit >> #initializePrimitiveTable` there will be all the entries for the jitted primitives and in `StackInterpreter >> #initializePrimitiveTable` the entries for the interpreter's ones.

So to *really* implement a primitive, besides implementing it we have to make an entry for it in those tables.

_Of course a primitive doesn't need to exist both as a intepreter primitive and as a jitted primitive. It's possible to only have one implementation of a primitive._

So, let's look for an empty spot in the tables to put our new primitive. For example position 231 seems unused, so let's put it there.

In `Cogit >> #initializePrimitiveTable` we add:
```
        ...
        "(221 genPrimitiveClosureValue	0)" "valueNoContextSwitch"
        "(222 genPrimitiveClosureValue	1)" "valueNoContextSwitch:"

        (231 genPrimitiveFormat 0)                                        <-- NEW LINE

        "SmallFloat primitives (540-559)"
	(541 genPrimitiveSmallFloatAdd				1)
```

And we do the same in `StackInterpreter >> #initializePrimitiveTable`:
```
                ...
                (230 primitiveRelinquishProcessor)

		(231 primitiveFormat)                                    <-- NEW LINE

		(232 239 primitiveFail)
                ...
```

Now our new primitive are ready to be used...

Are they?

##### Compiling the VM with the new primitive(s)

.....................



---- TO DELETEEEEEEEE -----  



Suppose we have the two implementations of a primitive to be introduced to the system. We have the interpreter one and the JITted one.

For example, the new primitive is `primitiveFormat`, so the corresponding  implementations  methods will be: `genPrimitiveFormat` and `primitiveFormat`

To (micro-)benchmark it we would need to compile 2 differents VMs: one with the JITted primitive and another one without it, with only the interpreted one implemented.

Now we test each VM separetely: 

First, we create an image. In it we need to *use* the new primitive. One way to do this is adding a method of the Object class.
```
Object >> format 
        <primitive: $number_of_primitiveFormat>
        ^self primitiveFail
```

Then, we also add the actual (micro)benchmarking logic. Again this could be done in a lot of ways, a simple one could be doing this in a Playground:

```
    [Object new format] benchFor: 5 seconds
```

#### [Sidenote about `benchFor:`: I suggest to just look at the implementation but basically this method executes the block given as a receiver for the time given as a parameter and then returns the average number of executions per second]

Once the image is set, we launch it using our 2 VMs and the compare the results.

How to compile each of the VMs?

For the one with the JITted primitive, first we should make sure of:
1) having the genPrimitiveFormat implemented.
2) adding it to the JITted primitive table. This is done in ``` Cogit >> initializePrimitiveTable ```

Then we open Iceberg and create a new branch, called for example `primFormat` [This step of course if not strictly necessary but strongly recommended]. Then we commit the new changes.

After this, everything from the Pharo side is already configured. Then we open a new Terminal (outside of the image of course xD). We move to the directory of the VM repository. If the primFormat branch is 
not automatically set, we set it accordingly.

There we compile the VM with the usual cmake command (and then the corresponding make install in the build directory).

Once the VM is generated, we launch the image with it.

For the not JITted one, it's enough to comment (or remove) the entry on the JITted primitive table. And then, follow the same steps as before, to generate a VM that has the
primitive only implemented in the intepreter.
