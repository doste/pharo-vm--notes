### How to micro-benchmark a new primitive? 

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