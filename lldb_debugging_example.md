### How to debug some PharoVM code with lldb?

Situation: We have a C function `foo_in_c` defined in some dynamic library called `test_lib`. We are using it in some way inside Pharo.
 It doesn't matter how it's used, the important thing for this example is we'll have to refer to this function, as it's defined outside Pharo, its address it's wrapped in an `ExternalAddress` object.

To call this function we'll need to know its address, so we can be pretty sure that we'll need to use this `ExternalAddress`.

Then, we have this `ExternalAddress` object, for some reason we want to check that the address that this object refers to it's correct, that is, that corresponds to the actual
address in the dylib.

An `ExternalAddress` object only contains one slot, corresponding to the actual memory address of the C function.

Let's suppose that we are already debugging the VM. So, inside the Terminal, we already in a `lldb` session.
For example, just where something related with the `ExternalAddress` object broke so the debugger steped in. 
Let's suppose that the problematic `ExternalAddress` object is a variable called `someOop`

Then, inside `lldb`, we would

```
lldb> p/x someOop

> 0xFFFFBEEF (the memory address corresponding to the object someOop)

lldb> p longPrintOop(0xFFFFBEEF)

> a TFExternalFunction ...
a ExternalAddress : 0xCAFECAFE
a TFFunctionDefinition ...
...
```

With this we can see what kind of memory layout the object has.

Supossing that the `test_lib` is already loaded by the VM, so all the symbols (global variables and functions) are visible to us.

We can check the memory address of `foo_in_c` in `test_lib`, like this:
```
lldb> p/x foo_in_c

> 0xEEEE0000
```

So, we want to see where this value appears in the `someOop` object:


We have the `ExternalAddress` object with address `0xCAFECAFE`, what we would need to do is access to this memory and see what's inside.

In `lldb` to read memory we use the 'memory read' or 'x' command:

```
lldb> memory read --size 8 --format x --count 1 0xCAFECAFE + 8

> 0xEEEE0000
```

This same command can be written like this:

```
lldb> x -s8 -fx -c1 0xCAFECAFE
```

This commands reads from the memory address specified, it will read one chunck (count=1) of size=8 and it will format it as hex (format=x)

You may be wondering why the + 8 to the address.

Remember about the memory layout of objects. All Pharo objects have a header defined in its first 8 bytes. So if we want to access the first slot of any object we would need to
skip the header.
That's why accessing the memory pointed by `0xCAFECAFE + 8` will give us the contents of the first slot, which is the actual external address.

                                                                                                            
                                                                                                                                                            

                                                                                                        
                                                                                                        
