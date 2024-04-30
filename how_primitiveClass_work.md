### How does the primitiveClass work?

 This primitive can be called in two different ways:
 
 #### 1)
 ``` MyClass >> class
        <primitive $NumberCorrespondingTo#primitiveClass>
 ```

 #### 2)
 ```
 MyClass >> classOf: X
        <primitive $NumberCorrespondingTo#primitiveClass>
 ```

 This means that the argument of the primitive can be either in the receiver or in the first parameter.

```
 primitiveClass
	| instance |
	instance := self stackTop.
	(argumentCount > 0
	 and: [objectMemory isOopForwarded: instance])
		ifTrue:
			[self primitiveFail]
		ifFalse:
			[self pop: argumentCount + 1 thenPush: (objectMemory fetchClassOf: instance)]
```

The `instance` variable is the stackTop, so for the case *1)* this would have the receiver and in the *2)* the first (and only) argument.
The method `isOopForwarded` checks that the `oop` (the variable `instance` in this case) is not an immediate and that it's a forwarder object. 
 To do this, it access the header of the `oop`.
If the 22 bits corresponding to the ClassIndex are 8 then it's a forwarder. 

You may be wondering why the `argumentCount > 0` before checking if the `instance` is forwarded. This first condition checks that `instance` is in fact the receiver or the first argument of the method.
If argumentCount is greater than 0 it means that then we do have an argument, so `instance` must be this value. In that case we have to check that is not a forwarded. 
If it is, the primitive fails. We can't ask the class to a forwarder. The forwarder job is only to forward all accesses. You can think of it as a `temporal` object. 
It will live until the accesses associated with it become patched. It doesn't make sense to ask it for its class.

And now you may be wondering why this forwarded check applies only if the `instance` is in the argument. 
This is because we are guaranteed that the receiver of the method is *always* resolved. This is, it's never a forwarder. So, we don't need to do the check :)
