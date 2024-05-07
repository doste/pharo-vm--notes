## What are forwarder objects?

Remember that all Pharo objects when looking at them from the VM perspective are just memory addresses. They are just a little block of data residing in the memory region that the VM controls.
 Often, the objects get moved to different locations in memory. In fact, this happens quite often. You can imagine that in a typical execution of a Pharo image there a lot of objects, so the VM has to handle all the consequences of moving the objects from place to place.

Here, we will concentrate in what happens with the other object that reference the object being moved.

So, if an object get moved in memory we need to track all the references to it and fix them up. One easy and direct way would be to to search for every reference to this object in the entire memory, one by one. Of course this would not scale. It would perform very bad.

The solution is to use *Forwarder Objects*

Suppose we have an object like this:

IMAGEN1

We have our object `One` that has two instance variables: `x` and `y`.
    Our `One` object has already been instantiated, so its two variables reference another objects.
    `x` references an object `Two` with three instance variables: `a`, `b` and `c`.
    `y` references an object `Three` with two instance variables:  `foo` and `bar`.

Then, object `Two` gets moved in memory. So, a copy of it is made in another part of the memory. The original `Two` object references its copy.

IMAGEN2

Now, the original `Two` object gets converted to a _forwarder object_. This imply having a complete different layout in memory. Now, the object has only one slot (an instance variable basically) that references (points to) the original `Two` object.

IMAGEN3

The _Forwarder_ stays like this until the first access: When the `One` object access its `y` instance variable it will want to acces to the `Two` object. But when it follows the reference from its `y` value it finds the _Forwarder_, and this one in turn will take its hand and take it to the `Two` object. That's exactly when the access is 'patched'. From now on, the `y` instance variable of `One` will reference the new memory address of the `Two` object directly.

The same logic would apply if there would be more than the `One` object referencing the `Two` object. 

When all the references have been 'patched' then the _Forwarder_ object is no longer necessary. So, it's ready to be garbage-collected.

Imagen4