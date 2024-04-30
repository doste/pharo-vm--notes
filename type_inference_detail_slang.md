 ### Small important detail about Slang:

 When we see that a method that looks like this:
```
    foo 
        | value |
        <var: #value type: 'unsigned int'>
        ...
```

 When Slang generates the C code from this method, the pragma at the beginning helps it by telling it that the variable called `value` must be of type unsigned int.
 If we don't add the pragma, Slang will try to infer its type and it may mess everything up.