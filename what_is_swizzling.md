### What is swizzle

*Swizzle* means to patch pointers when the image get initialized/loaded.

At the moment we save the image, of course we have to save everything. This includes all the _live_ objects. These objects are all stored in the heap memory (it's not actually the real heap of the OS process, for example the objects don't get allocated with a call to `malloc` but conceptually we can think of this region of memory as _a_ heap).
 So, all objects come down to pointers to this memory. This means that if we want to save all the _live_ objects we have to save all the pointers that correspond to them.

Of course we don't have control over which memory address a given pointer points to. Again, the objects are *not* allocated by using `malloc` but they are allocated in a similar way. 
So it may be easier to think about the allocation as just using `malloc`. The important thing is that we don't choose the memory address for a given object. When we allocate space for it, we would get a pointer and that's it.

When we save the image, it would make much sense to save all these pointers directly. When we would load the image again, all those addresses maybe don't even correspond to a valid memory region. Maybe our _"heap"_ now it's in another place.

So, what the VM does, is save all those pointers but serialized. The values of each pointer get saved but also an absolute memory address which will work as a base.
Then, what will be important about each pointer value will be the offset they have with respect to that base.

To *swizzle* the pointers means doing that patching. To each pointer, give it a new address based on its offset with the base.

