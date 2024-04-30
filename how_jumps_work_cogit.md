## How jumps work in Cogit?

A jump instruction always needs a target, this can be any address or label (actually a label only references some memory address).

There are two types of jumps: Backward and Forward.

- Forward case:
    When we are writing a Cogit method, we are actually writing the first stage of the compilation process. We are writing code that then Cogit will generate the actual machine code from.
    When executing a method like a genPrimitiveX for example, line by line each message send will generate the corresponding assembly equivalent.
    But, when it finds a Forward Jump, the target place, the memory address that it wants to jump to, is hasn't been visited yet. We are going line by line!
    So, in that case, it will simply generate a Jump with target: 0 and then we eventually find the actual target address, it 'patches' the target. Now that that address has been visited.

    Example:
````
    ...
    jumpNotImm := cogit JumpNonZero: 0
    ...
    ...
    jumpNotImm jmpTarget: cogit Label
````

    A small note about the `cogit Label:` this references the same address where it's defined. So, the target of that Forward Jump will be the exact address where the jmpTarget is set.


- Backward case:
    For this case, when we visit the Backward Jump, we know that the target has been already visited. So in this case there is no need to patch anything. The target can be set accordingly 
    at the time the Backward Jump is visited.
