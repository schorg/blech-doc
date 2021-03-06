---
title: "Local and external variables"
linkTitle: "Variables"
weight: 20
description: >
  This explains the difference between local and external variables in code generation.
---

## Local variables

Variables declared with the scope of an activity are persistent.
They keep their value over multiple reactions.
Hence, in the generated code, they are stored outside the step-function of the given activity.
Instead they are passed as a context with every call of the step-function.

Previous values however need not be stored in this static context.
In the generated code, they can be automatic variables (stack allocated variables) of the generated step-function.
The step-function itself will create the prev variables (where needed) and fill them before the actual step is carried out.

Special attention has to be given to the reaction in which the declaration of the local variable takes place.
Here the local variable assumes its first value.
This is defined to be the prev value and thus the prev variable has to be reset here once.

```blech
...
var x: int32 = 7 // prev x = 7
cobegin
    run B(prev y)(x) // x may have changed
with
    run A(prev x)(y) // prev x is guaranteed to be 7 in the first reaction
end
...
```

## External Variables

External variables (inside the activity scope) are dual to local variables in terms of code generation.
We do not need to store external variables in the static context which is passed to the step-function.
Instead an external variable will be represented by an automatic variable which is set at the beginning of the reaction using the value of the actual external C-variable that it is bound to.
At the end of the reaction, the final value is written back to the external C-variable (if it is a `var`).

Previous values of external variables on the other hand must be persisted in static memory.
This is to guarantee synchronous semantics, i.e. the prev value really is the value that the last reaction ended with 
(while the value of the external variable is volatile and may have changed arbitrarily between reactions).
The code generator will create an extra local activity variable and add it to the list of local variables that are stored in the context.
The value of the prev variable will be set *at the end* of each reaction, so it can be used in the next reaction.

Again, special treatment is required in the code block that represents the reaction in which the declaration of the external variable becomes active.
Here, as before, the prev value has to be set once so it is defined in the case that the prev value is used in the same reaction.

## Effects on compilation contexts

If necessary, the function `ActivityTranslator.cpAction` adds the external variable with a `prev_` infix in its name to the `Compilation.iface.locals` of the current activity.

`ActivityTranslator.makeActCall` transports the locals of the called activity to the interface of the current one which includes the prev'ed version of external variables.
In order to be able to print this interface as C code, the variables are added to the dictionary of known declarations in the type check context (`ctx.tcc.nameToDecl`).

Since the entry point activity is never called from within the Blech program the above mechanism of adding variables to the type check context is repeated in `ActivityTranslator.translate` (for the entry point activity only).

## Effects on C-code generation

Printing external constants or params does not require any additional modifications. The given binding is simply inserted. For `extern let` and `extern var` additional effort is required.

1. External variables

   Whenever an external variable is accessed as a right-hand-side, left-hand-side or the input or output argument to a call, it needs to be rendered differently than local activity variables.
   This is because external variables are generated as automatic variables in the scope of the step-function (instead of being passed in with the context).
   Thus they need to be rendered like local variables in Blech functions.
   This is achieved by hooking in the `isExtCurVar` function in the call of the `cpRenderData`.
   It will set the `subProgDecl` to `Function` if the variable name to be rendered is an access to the current memory location of an external variable.

2. Access to previous locations of external variables
  
   Usually the previous value is stored in an automatic variable of the step-function and is accessed accordingly. The name is mangled by prepending a `prev_` to the QName of the variable.
   However previous values of external variables are stored in the context. Thus they need to be accessed like a local variable. The `prev_` infix is already mangled into the QName that is stored in the context.
   Thus `cpRenderData` will change `timepoint` to `Current` and add the `prev_` infix into the name to be rendered.

3. Copy-in / copy-out 
   
   The function `ActivityTranslator.translate` generates code for
   
   * Copying the values from the bound external C-variables to the local memory inside the step-function at the beginning of the reaction.
   
   * Copying back the final results back to the bound external C-variables at the end of the reaction.
   
   * (If necessary,) setting the previous memory to the final current value of the external variable such that in the next reaction the prev value can be used.

## Effects on JSON trace printing

Since prev locations of external variables become local activity variables and are added to the type check context, they are printed in the trace along with the normal variables.
Note however that the value shown in the trace is the prev value for the *next* reaction.