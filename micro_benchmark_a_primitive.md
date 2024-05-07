# How to create a new primitive and (micro-)benchmark it?

In this tutorial we'll see how to implement a new primitive. 
 To do that we'll implement a concrete real (even a useful! :P) primitive. We'll see all the steps we have to go through to not only implement it but also test it and (micro-)benchmarking it. We'll also see all the components involved in the process and how they interact with each other.

A prerequisite to be able to follow and code along this tutorial is to have all the VM code downloaded and installed. See [1] below if you need help with this.

 ## primitiveFormat
 
The primitive we are about to implement gives us the format of the class of a given object.

### The format of an object

All objects in Pharo have a 'Header', this gives us all kinds of information about the object. The VM is the component that is the most interested in this data. From a user point of view (from the image side for example) the header of an object is not even accessible.
 In this header, among other things, there is a field called the *format*: This value describes the layout in memory of an object. In a 64-bit VM, the header of an object has 64 bits. 5 of these 64 bits are used for the format.
 For this tutorial that is all you need to know about the format of an object, but if you are curious see:
 - [A presentation made by the PharoVM team about the layout of objects](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-09-29-VM-Hernandez-objectsLayout.pdf)
 - `Behaviour >> #instSpec` : The comment in this method describes each format.

 ### Back to primitiveFormat:

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


 ### Implementing IntepreterPrimitives >> primitiveFormat

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
When the primitive is executed we have the receiver in the top of the stack, this receiver is the one who "called" the primitive. We check that the receiver is not an immediate object [2]. [Explanation in the end]

 Then we extract the header from it, and then from the header the format. Finally we pop the receiver and pushed the result, this is the format of the receiver. (argumentCount is 0 in this case, this primitive doesn't take any argument).

Pretty simple, right? Most of the work is done by the helper methods that give us the header and the format of the object.

Now, let's move to the jitted one.

### Implementing CogObjectRepresentation >> genPrimitiveFormat

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


### Installing the primitives

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

### Testing the new primitive

We could go directly to compile the VM with new introduced primitives but as good software engineers we should test them first.  It's a good practice that every time we introduced a significant change in the system, we also add tests that confirm (not 100% of course! :P) that they work.

#### How to test a primitive?

If we go to the `VMMakerTests` package we can see that there a lot of classes there, to test all the components of the VM.
As we have the two implementations of the primitive (the interpreter one and the JIT one) we'll have to write tests in different classes.

##### Testing primitiveFormat

We'll start with the interpreted implementation. We can look at all the classes in the package to see where other primitives implemented by the interpreter are being tested. We can find the `VMPrimitiveTests` class. This one seems the correct place :)

A good rule to follow when we don't even know how to start writing such a test is to just check what the other tests are doing. Even copy-paste a little, just keeping in mind what we want to test.

<ins>Important detail about testing components of the VM:</ins>

Here we are testing the VM itself, but we are inside an image that is running in a (most certainly) production VM. This may sound confusing, but we are writing components of a VM but of course we are standing on another VM that is the one running the image we are in.
 To be able to use this VM we are developing we'll have to compile and build it, but that's the next section. Here we want to test it. To do so we need a simulated environment, because the VM deals with _Oop_s and not with Pharo objects.

-----------
###### What is an Oop?

They are basically how the VM sees the objects it manipulates. You can think of them as the low level representation of a Pharo object. In an _Oop_ it's described how a Pharo object is layed out in memory. 
 If you are familiar with CPython, an _Oop_ is similar to a _PyObject_. 
 It's the underlying implementation for all objects.

 Again, if you are curious refer to:
 - [A presentation made by the PharoVM team about the layout of objects](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-09-29-VM-Hernandez-objectsLayout.pdf)

-----------

So, back to testing the VM: 
  When we are testing a component of the VM, we can't give just a Pharo object. It won't know what to do, it won't we able to handle it. The VM works with memory address. For the VM, an object (an _Oop_) is just a memory address, so to manipulate it, it will try to access it and potentially dereference it to access other _Oop_s referenced from it. (For example a instance variable of an object is another object, so you can see how a chain of references is formed when dealing with _Oop_s).

What we do in the tests is to simulate the environment. This is, we create objects that are not really Pharo objects but _simulated Oops_ that reside in a _simulated memory_.

For example to create a _simulated_ `Array` object we use the following code:
```
object := self newObjectWithSlots: 0 format: memory arrayFormat  classIndex: memory arrayClassIndexPun.

```
This method will create an object but in the _simulated memory_, then when we want to test a component of the VM, we give it this memory, so it will be able to operate on it because the objects are just (simulated) memory addresses, just as the VM expects them to be :)

All the simulation of the memory for tests is handled by the `VMSpurMemoryManagerTest` class. Check out all the methods it has to know all the things it can do.

Ok, let's then write a test:
```
testPrimitiveFormatWithArray
	| object |
	object := self
		          newObjectWithSlots: 0
		          format: memory arrayFormat
		          classIndex: memory arrayClassIndexPun.

	interpreter push: object.
	interpreter primitiveFormat.

	self deny: interpreter failed.
	self
		assert: interpreter stackTop
		equals: (memory integerObjectOf: memory arrayFormat)
```

 If you look closely to how we instantiate the object, we are giving it the format as one of the arguments, so it's pretty straightforward what we want to test:
 We just want to check that the value returned by the primitive is the same as the second argument of the `newObjectWithSlots:format:classIndex:` :D

 So, we create the (simualated) object, then invoke the primitive. Remember that in the intepreter all the arguments are passed by the stack (the interpreter doesn't even have registers!), so to pass the object to the primitive we just push it to the stack. 
  Then, the callee (in this case the primitive) will have to pop all the arguments and push its result. So, the result of a method (or primitive) will be always at the top of the stack when returning.

 Then we assert that: the interpreter didn't fail executing the primitive and that the value returned is the expected format. Done! We can run the test by clicking in the little circle besides the name :D

 Try to write your own tests too! [You may want to check the methods that end with _format_ in the `SpurMemoryManager` class].


##### Testing genPrimitiveFormat

Now, to the jitted one.
 Again, in the package `VMMakerTests` we should look for where the jitted primitives are being tested. There is a `VMJittedGeneralPrimitiveTest` class, this seems a nice place to test our new primitive.

Let's try to write a similar test to the one we wrote for the interpreted primitive. We'll test the `genPrimitiveFormat` with an `Array`.

Now, if you thought the simulated environment for the test we wrote was a little wild, brace yourself because it's going to get wilder xD

Remember that when we _JIT_ (as a verb) a method, including primitives, what we are essentially doing is generating machine code during run-time that behaves (hopefully) exactly as if we would execute it in the interpreter. So, to _JIT_ a primitive means generating machine code from it. 
 Then, if we want to test our `genPrimitiveFormat` we'll need to test machine code. And if we want to test machine code we'll need a processor to understand that code.
 It would be quite painful and counterproductive to have our computer real processor to run each of the methods we want to JIT. Again, the magic word: _SIMULATION_ 🌈

We just simulate the CPU processor. Then, the generated machine code by the JIT is given to this _simulated processor_ so it executes it and then we check the results.
Pretty nice, right?

-----------
###### Unicorn

The CPU simulation is all handled by the `UnicornProcessor` class which uses the [Unicorn Engine](https://github.com/unicorn-engine/unicorn) behind the scenes.
For this tutorial is not necessary to know much about `Unicorn`, just enough to be able to use it :)

When testing a jitted method, there are mainly two kinds of operations that we want to perform using the `Unicorn` simulator:
- To run some machine code, from a given address to another given address.
- To ask for its state, most often the values of its registers. In the tests, we'll often want to assert about the values of the registers after running the code.

To use the simulator, our Test Class already inherits from `VMSpurMemoryManagerTest` and this one has a instance variable already set with the `Unicorn` simulator.
It's called `machineSimulator`.

That's pretty much all you need to know to use the simulated CPU on our tests :D

-----------

###### Back to testing genPrimitiveFormat

We can write a test like this:
```
testPrimitiveFormatArray
	| primitiveAddress |
		
1	primitiveAddress := self compile: [ 
		cogit objectRepresentation genPrimitiveFormat ].

2	self prepareStackForSendReceiver: (memory newArrayWith: {42 . 123 . 31}).

	"Assert it reaches the caller address and we have the format in the receiver register"
3	self runFrom: primitiveAddress until: callerAddress.
4	self assert: self machineSimulator receiverRegisterValue equals: (memory integerObjectOf: memory arrayFormat).
```
In the line 1 we are telling to the JIT to generate the machine code corresponding to the `genPrimitiveFormat` we already wrote.

In line 3 we tell the CPU simulator to run the generated code for the primitive. But remember, our primitive receives an argument! And that argument it receives it through the stack. So we need to model exactly that in our test. In line 2 we prepare to stack to give it to the primitive.
 Again, remember we are in a simulated environment so we can't just write `Array new:...`. We have to create a _simulated Array_.

 In line 4 we do the assertion. We want to check about the _return value_ of the primitive. We want to assert that it's correct, that is it's the format for the `Array`.
(Check the implementation of `genPrimitiveFormat`, the result is put in the `ReceiverResultReg` register).


Ok, now we have confidence that our primitive works. At least in the simulations. We should test it in a _real_ environment, this is: to launch a new image that uses our new primitive.
 To do that we'll need to lauch that image using the VM we are developing right now, the one that implements the primitive `format`. So, we'll need to compile this new VM.


### Compiling the VM with the new primitive

All the work we have done we did it inside an image (of course) that is running in a VM (most likely the one integrated with the *Pharo Launcher*). We have written code of the VM itself, but use it (outside of the simulations) we need to compile it! So, we need to build a new VM from all the code we have written. 

#### First step: Committing the changes

Open Iceberg, go to the repo containing the VM code. Then, we should create a new branch, called for example `primitive-format` [This step of course if not strictly necessary but strongly recommended]. Then we commit the new changes.
 We should see all the changes we have made in the tutorial, they are all in 6 classes: `VMJittedPrimitiveTest`, `VMPrimitiveTest`, `CogObjectRepresentation`, `InterpreterPrimitives`, `Cogit` and `StackInterpreter`.

After this, everything from the Pharo side is already configured. Then we open a new Terminal (outside of the image of course xD). We move to the directory of the VM repository. If the `primitive-format` branch is not automatically set, we set it accordingly.

#### Second step: Building the VM

There we compile and build the VM using the commands describes in the [PharoVM Wiki](https://github.com/pharo-project/pharo-vm/wiki/General-Build-Information):
```
$ cmake -S pharo-vm -B buildDirectory
$ cmake --build buildDirectory --target install

```
----------------------
If you are already familiar with `cmake` feel free to skip this section :)

The argument for the `-S` flag in the cmake command indicates the 'Source Directory', if you already in that directory you should just write a '.', like this:
```
$ cmake -S . -B buildDirectory
```
The argument for the `-B` stands for the 'Build Directory', this is where the "output" of the cmake build process will be put. 
Again, if you in the directory where the source is, you should put the build directory outside, one level up. For example like this:
```
$ cmake -S . -B ../buildDirectory
```

The second `cmake` command is to actually build the VM from all the sources generated by the first command. Here the argument for `--build` should be exactly the directory that you chose as 'Build Directoy'. For example like this:
```
$ cmake --build ../buildDirectory --target install
```

----------------------

#### Third step: Running an image in our new VM

After `cmake` is done, we should get an executable file: our new built VM! :D

This executable will be inside the `buildDirectory/build/dist` directory. If you are in the Terminal, check with the `file` command to make sure that that `Pharo` file is in fact an executable file.

For example if you are in Mac, you should see this output:
```
... /build/dist/Pharo.app/Contents/MacOS/Pharo: Mach-O 64-bit executable x86_64
```
(In Linux it would be an ELF file and in Windows an EXE).

Oh right, we have our VM ready to run. But a PharoVM in its own can't execute anything, we need to give it an image.


##### Preparing an image to use the new primitive

The work on the VM itself is done. What's left is to make use of the new primitive (and not in a simulated way xD). To do so, we need to do it in an image. 
The recommended way is to create a new image from scratch, then write in there the code that will call the primitive.

So, let's open the `Pharo Launcher` and create a new image with a good name: for example, `Pharo11_to_use_primitive_format`. 

In this new image, we need to add a method that calls into our new primitive. We can call said method just `format`. In what class should we put it?
_Every_ object has a format, so the `Object` class seems like a good place. We know every Pharo object inherits from this class.

```
Object >> format 
        <primitive: 231>
        ^self primitiveFail
```
_In the number after the colon we should put the index we chose for the primitive in the `initializePrimitiveTable` methods. If you choose a different number make sure here to put it_

Then, we can try this new method. For example in a Playground. Some examples to try:
```
    Array format.
    String format.
    WeakArray format.
```
Experiment with other Objects to see how their format differ!


### (Micro-)benchmarking a primitive

Now, in our final step of introducing a new primitive to Pharo we would like to see how it performs. We have two implementations of the primitive `format`: the one with interpreter and the jitted one.

Let's say we want to know how much performance we gain by using the jitted one. So, we want to know how much difference in performance are there when only the interpreted one is being used against when using the jitted one.

 To properly do that we'll need two VMs: one with only the interpreted primitive and the other with jitted one too.

 _Small detail: it doesn't make sense to have only the jitted one. The system just doesn't work like that. The interpreter has to be able to execute eveything. There is no such a thing as "this method is only jitted, not executed by the interpreter". It's the other way around: The interpreter can execute everything but the JIT cannot, there are certain methods and primitives that are not implemented in the JIT_

So, to build the two VMs we follow the same procedure as before, *but* to build the one with only the interpreter implementation of the primitive we need to do a small tweak:
  It's enough to comment (or remove) the entry on the JITted primitive table (the one in `Cogit >> #initializePrimitiveTable`). And then, follow the same steps as before, to generate a VM that has the primitive only implemented in the intepreter.

Ok, we now have the two VMs. In the build process (`cmake` commands and all that) we should specify different names for the build directory, so now we end up with two different executables.

Again, we launch an image from them. Inside that image we'll add the actual (micro-)benchmarking logic. 
Again this could be done in a lot of ways, a simple one could be doing this in a Playground:

```
    o := Object new.
    [o format] benchFor: 5 seconds
```
---------------
Two little remarks about this code:

- Sidenote about `benchFor:`: I suggest to just look at the implementation but basically this method executes the block given as a receiver for the time given as a parameter and then returns the average number of executions per second.

- Look at how the object creation/instanciation is outside of the block to benchmark. This gives us the ability to focus on the logic that we want to bench (the `format` method), so the object instanciation doesn't add noise in the performance metric.

----------------

So, to compare the implementations of the primitive `format` then we launch the same image (the one that has the code that implements the `format` method and has a Playground with the previous code) and compare the results of the `benchFor:` method :)

That's it! We implement a new primitive, test it and then build a new VM with the primitive implemented. Pretty cool, right? :D


[1] Installing the PharoVM
  What we want is to have all the code of the PharoVM in the image we are using. If we don't do this we would not have access to for example the primitives table.
So, it makes sense that if we want to hack on the VM, well we need the code of the VM! xD

First, follow the steps in the [PharoVM Wiki](https://github.com/pharo-project/pharo-vm/wiki/General-Build-Information).
Once you have the directory with all the code you need to bring it to the image.

Open Iceberg. Click on *Add*. Then *Import from existing clone*. There you look for the directory where the code is.
[This directory should be the 'top level' one, not the build one. If you follow the steps exactly as they are described in the Wiki, its name should be `pharo-vm`]

In Iceberg, not it should appear in the list of Repositories. It should say 'Not loaded'. To load it: Right click on it -> `Metacello` -> `Install baseline of VMMaker (Default)`. After a few minutes you should now have all the VM code inside the image. 

To be sure look to the `VMMaker` package in the Browser.


[2] Immediate objects.

The address of an object in memory is used to make a reference towards it. Sometimes, for performance or memory consumption reasons, instead of an address, a reference is used to represent another value. A tagged value is a value encoded inside a reference. Tagged values are used to represent some immediate objects such as SmallIntegers and Characters. The Pharo VM only stores object references whose addresses are aligned. It means that some of the last significant bits of reference cannot be set to 1.

For example, in 32-bits, possible object addresses are multiple of 4 bytes: 4, 8, 12, 16… Respectively, the VM never stores unaligned addresses such as: 1, 2, 3, 9, 10, 11… In binary, aligned addresses are as follows:

```
#(4 8 12 16) collect: [ :n | n storeStringBase: 2 ]
>>>  #('2r100' '2r1000' '2r1100' '2r10000')
```

We can see that addresses aligned on 4 bytes always have the last 2 bits (2^2=4) equals to 0. On 64 bits, addresses are aligned on 8 bytes and we have the last 3 bits (2^3=8) that are always equal to 0.

In other words, in the Pharo VM, a tagged value is just a value of 1 word length whose the least significant bits indicates the type of value that is encoded. The VM uses either the two least significant bits of a reference to support tagged values in 32-bits or the three least significant bits in 64-bits. This means that 30 bits are free to encode a value in 32-bits and 61 bits are free in 64-bits. In the Pharo VM tagged values are used to represent some immediate objects such as SmallIntegers and Characters.

Read more: [Booklet-PharoVirtualMachine](https://github.com/SquareBracketAssociates/Booklet-PharoVirtualMachine/tree/master)
