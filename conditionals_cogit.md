### How to write conditionals in Cogit?

We want to execute the code in `[if_body]` if the value in the register `r` is the same as the (quick) constant `X`. If they are not equal, then we execute the code in `[else_body]`:
```
    cogit CmpCq: X R: r
    jumpNotEq := cogit JumpNonZero: 0.

    [if_body]

    jumpNotEq jmpTarget: cogit Label.

    [else_body]
```