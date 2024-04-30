
### JITCompilation of a CompiledMethod

A `CompiledMethod` get JITCompiled as a whole (x)or it doesn't.
If it *does* get JITCompiled, that means that every bytecode that the method contains must get JITCompiled. For that, every bytecode will invoke its corresponding `genBytecode...` method. That will generate Cogit code. It will not execute anything by itself, just generate code.
 For example, inside the JIT implementation of the new FFI bytecode `genBytecodeFFICall:` there is something like this:
  
  ```self CallR: someRegister```

This line will *not* call anything, it will generate code that when executed it will do the call. 

Every one of the methods that start with `gen` are the ones corresponding to the JIT, so all of those will generate an IR that eventually will be 'concretized' to "real" machine code, for example x86.

But, it's important to notice that each bytecode gets JITCompiled by itself, like a little piece of a (maybe giant) puzzle that is the `CompiledMethod` containing them.
After all bytecodes are JITCompiled, just there it's that they are kind of merged with each other to form a `'JITTED' CompiledMethod`.

You may be wondering how the merging of bytecodes work, how the bytecodes communicates with each other so that the final result makes sense.
 The answer is that they have a calling convention, and this is how they communicate.

  #####  The caller pushes the arguments, the callee pops them and then push the result. 

This of course apply to every bytecode, so that's how they can fit nicely when merging them.


And that's why when we write a `gen...` for a bytecode, at the end the result has to be pushed. It doesn't have to be at a specific register nor we need a `return` instruction, this is because after a bytecode it's supposed to come another bytecode and so on. 

But, what if at the end of the whole `CompiledMethod` we *do* want to return? Well in that case, we would have a specific bytecode to model that behaviour, which in turn, when JITCompiled it will generate the `return` instruction.

To where can we return from a bytecode you may be asking.

Dealing with JITCompiled code for a bytecode there are two situations where we would need a `return` instruction:

-  If we were called from another bytecode, that is also JITCompiled, so its code is essentially machine code.
   This scenario is basically like a giant assembly file, where each bytecode calls each other like they are simple assembly functions. In that case, of course to return to the caller we need a `return` instruction (the caller called us using a `call` instruction).

- In the trampolines: The interpreter was executing some (intepreted) code, we then jump to some JITCompiled code, then to go back to the interpreter we would need a `return` instruction.
    