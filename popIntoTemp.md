## How does popIntoTemp: work?


Suppose we have a class OurClass with these instance methods:
```
someMethod
	| someTempVar |
	someTempVar := self someOtherMethod: 10.
	self printResult.                           "printResult doesn't matter"

someOtherMethod: anArgument
	^anArgument * 9
```


If we execute the following:

`(OurClass >> #someMethod) symbolicBytecodes`

for example in a Playground, we get the (symbolic) bytecodes corresponding to someMethod.

Here they are:
```
    49 <4C> self
    50 <20> pushConstant: 10
    51 <91> send: someOtherMethod:
    52 <D0> popIntoTemp: 0
    53 <4C> self
    54 <82> send: printResult
    55 <D8> pop
    56 <58> returnSelf
```

If we look at the body of someMethod we see that the result of sending the message `someOtherMethod:` with argument 10 is stored in the temporary variable `someTempVar`.

###How does the bytecode model that behaviour?

First we have to push the argument, it's a constant so we use `pushConstant:`. By doing this we are passing the argument to the method we are about to call.

Remember that the calling convention of the bytecodes is that they pass their arguments by pushing them, and then the callee has to pop them and push the result.

Now it will be responsability of  `someOtherMethod:` to pop the (only) argument that we have pushed and then push its result.
We know for certain that after `someOtherMethod:` is done and the execution has return to us, in the top of the stack is the result of the message we have just sent.

We want to store that result in a temporary variable. To do that we have the bytecode `popIntoTemp:`

Well, finally how does `popIntoTemp:` work?

As many bytecodes (and primitives) there are two implementation of this bytecode: the interpreted one and the jitted one.

The interpreted version of `popIntoTemp:` is not that difficult to understand, the implementation can be found in:
`StackInterpreter >> storeAndPopTemporaryVariableBytecode`


The jitted version seems more tricky, so let's dive in:

There are two versions of the JIT compiler: _SimpleStackBasedCogit_ and _StackToRegisterMappingCogit_.
 As the names implie, the first one operates on the stack, basically accessing memory with each operation.
 The second one tries to optimize those access by moving the values to registers whenever it can. 

As you can imagine, every jitted method (be it a primitive or a bytecode) will have its own corresponding implementation, one using only the stack and other trying to move
everything it can to registers.

First let's look at the _SimpleStackBasedCogit_ implementation:

```
genStorePop: popBoolean TemporaryVariable: tempIndex
	<inline: false>
	popBoolean
		ifTrue: [self PopR: TempReg]
		ifFalse: [self MoveMw: 0 r: SPReg R: TempReg].
	self MoveR: TempReg
		Mw: (self frameOffsetOfTemporary: tempIndex)
		r: FPReg.
	^0
```
The argument popBoolean determines if we want to only read the value at the top of the stack or pop it too (this is removing it).
If we want to just pop it, we use the `PopR:` method. Else, we have to do a 'store' from memory to a register. We can do that with `MoveMw:r:R`. 


####[ IMPORTANT thing about Cogit: all these methods we see here like Pop's and Move's are not actually methods that simulate the instructions to execute, for example when the method
genStore:TemporaryVariable: is invoked, it will generate the corresponding assembly code. *Not* execute it!]


So, then `MoveMw:r:R` will generate code that behaves like this:
The first argument is an offset, the second a base and the third the destination register.
Essentially the generated code will read a word (that's why the *w* in `MoveMw:r:R`) from the memory address given by base + offset and then stores what's in there to the
destination register. 
 As you can see the base is the SPReg, this register is the 'stack pointer register'...

 ####[Remember that in Cogit all register are virtual, like the instructions themselves they will have to go thorugh a step called 'concretize' 
 which will convert everything to specific 'real' instructions and register, depending on the architecture]


... so, when we read from the memory pointed by this register what we get is the value at the top of the stack. The offset is just a number to add to this base, for example if we would want to access to the value below the top of the stack, we would need as offset the length of a word.
 It's important to notice that in this case we are not removing that value from the stack (we are not poping it), just reading it.

Then, wheter we pop it or just read it, we move it to the TempReg register.

So, now in TempReg we have the value at the top of the stack, as the name `genStorePop:TemporaryVariable:` implies we want to copy that value to a temporary variable.
To which temporary variable? Well, for that we have the second argument: it's an index. This index will determine what temporary variable we want to access.

So, we need to move the value that is in TempReg to whatever memory address the temporary variable has.

To do that, we'll do a 'load' from a register to memory. By using the method `MoveR:Mw:r` we move the value in the register given by the first argument to the memory address given by base + offset where base is the third argument and offset the second one.

In this case, base will be the FPReg, this register is the 'frame pointer register'. It points to the memory corresponding to the current frame. This is basically the activation context of a method. Every time we execute a method a new frame pointer is created. 
 Among other things, in the frame pointer is where all the temporary variables are defined and stored. In particular, they are defined at the top of the frame. 
To know exactly at what offset a given temporary resided, we have a helper method: `frameOffsetOfTemporary:`.

 With this we obtain the offset of the temporary variable and then by using the FPReg as base we can get the exact memory address where this variable is stored.
 We move the value in TempReg to this address. We are done :D


Now let's see the `StackToRegisterMappingCogit` implementation:

```
genStorePop: popBoolean TemporaryVariable: tempIndex
	<inline: false>
	| reg |
	self ssFlushUpThroughTemporaryVariable: tempIndex.
	reg := self ssStorePop: popBoolean toPreferredReg: TempReg.
	self MoveR: reg
		Mw: (self frameOffsetOfTemporary: tempIndex)
		r: FPReg.
	(self simStackAt: tempIndex + 1) bcptr: bytecodePC. "for debugging"
	^0
```
