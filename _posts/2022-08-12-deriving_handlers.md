---
layout: post
title: "Automatically deriving typeclass instances"
subtitle: "A dive into Lean 4's metaprogramming"
---

## What are deriving handlers?

Lets begin by looking at a very simple example of this metaprogramming tool Lean has implemented. If the user defines an enumerated data type with the keywords `deriving BEq`

```lean
inductive test where
  | con1 : test
  | con2 : test
deriving BEq
```

then Lean will somehow (as if by magic) derive an instance of `BEq test` automatically, and expressions like `con1 == con2` can be evaluated

```lean
open test

#eval con1 == con2 -- false
```

We may hope that we could leverage this kind of functionality for custom typeclasses we write ourselves. For example consider this bizarre typeclass which returns the name (as a string) of its type:

```lean
class MyName (α : Sort u) where
  nameID : α -> String
```

One could hope that we would be able to write `deriving MyName` to any declaration of an `inductive` or `structure` and it would Just Work™. Of course we know better. How could Lean possibly intuit our intentions about the functionality for `MyName` unless we tell it exactly what to do (though we test it anyway, just to be sure Lean can't read minds).

We've seen this work for typeclasses like `BEq` above, and others like `Inhabited`, `ToJson`, ... so what's going on there? Each of these typeclasses has a custom "deriving handler" that derives these instances automatically (or at least tries). These deriving handlers are metaprograms written in Lean itself. For example, check out some of the handlers implemented in core Lean: [/src/Lean/Elab/Deriving/](https://github.com/leanprover/lean4/tree/master/src/Lean/Elab/Deriving).

The goal of this guide is to provide a journey through Lean 4's metaprogramming in order to uncover what kind of metaprogramming magic is going on for these handlers, and what we have to do to implement one ourselves. 

## Looking at examples

In the [/src/Lean/Elab/Deriving/](https://github.com/leanprover/lean4/tree/master/src/Lean/Elab/Deriving) folder there's a few examples we can look at to draw inspiration from. What all these handlers have in common is that our aim is to produce something of type 

```lean
def DerivingHandler := (typeNames : Array Name) → (args? : Option (TSyntax ``Parser.Term.structInst)) → CommandElabM Bool
```

In essence we should expect the `DerivingHandler` to make some modifications to the environment (for example, evaluating a command that adds a particular typeclass instance) and return a boolean on whether it is successful.

After this some `DerivingClassView` method `applyHandler` can be invoked, and the handler is added to the environment via the  

```lean
initialize
  registerDerivingHandler ..
```

command. Cutting a long story short (the story is quite long, in fact the last section constitutes of some notes I took while I looked into the source), the path forward is clear: We need to write a function with type signature `Array Name -> CommandElabM Bool` that generates the instances we want for each data type named in the array. Once we do this we can invoke some magical incantation, and the type class can be derived for us.

In principle because we are metaprogramming geniuses after reading the metaprogramming book, we should be able to write this one out by hand. But to play it safe maybe the best place to start is by looking at some of the examples of handlers already implemented in Lean. In fact, the `FromToJson` handler is pretty close to what we want, so we can take a peak at the implementation there to get a sense for the general strategy.

The `FromToJson` handler is not too dissimilar from the other handlers in the same folder. Most of the instance handler files can be split into 3 main tasks, and for our organizational purposes we might as well split the 3 tasks into the three main functions for the module. 

### Make the Instance Handler

In order to invoke

```lean 
initialize
  registerBuiltinDerivingHandler `MyName mkMyNameHandler
```

We need to define the handler:

```lean
def mkMyNameHandler (declNames : Array Name) : CommandElabM Bool := do
  let cmds ← liftTermElabM $ mkMyNameInstanceCmds declNames[0]!
  cmds.forM elabCommand
  return true
```

This handler doesn't do much. It simply elaborates a number of commands that are defined by `mkMyNameInstanceCmds`. 

### Define the necessary functions

Luckily there are a bunch of useful functions defined in [Lean.Elab.Deriving.Util](https://github.com/leanprover/lean4/tree/master/src/Lean/Elab/Deriving/Util.lean) to make our lives easier. The API that exists around making instances is defined in this file, and the preferred method can be summarized in the following steps:

  1) For each inductive in a mutual block, generate a `Context`
  ```lean
  structure Context where
  typeInfos   : Array InductiveVal
  auxFunNames : Array Name
  usePartial  : Bool
  ```
  via `def mkContext (fnPrefix : String) (typeName : Name) : TermElabM Context`.
  This context includes all the information we will need about the inductive's constructors, and a list of auxiliary names we should use to name our functions.
  2) Define the actual functions that are needed in the definition of the typeclass. For example, for our typeclass `MyClass` and the function will take the form 
  ```lean
  def ~auxFunName~ ~binders~ : String  
  ```
  The auxiliary function name is luckily provided for us in the `Context` generated above. Not only does it generate a fresh name with no conflicts, but by using the name from the generated context we can refer to it in other parts of our program via its index in the array `auxFunNames`.

  The binders can also be generated via a handy pre-written utility. In this case, we call
  ```lean
    def mkHeader (className : Name) (arity : Nat) (indVal : InductiveVal) : TermElabM Header 
  ```
  for each of the functions making up the class (in our case, just one). `mkHeader` generates a `Header` (duh) from 
    * `className`: The class we are trying to derive 
    * `arity`, the arity of the class function
    *  `indVal` the inductive we are defining it on.
  In addition to the binders, the `Header` class has a number of other useful fields that we can use to generate the function (which we do not need in this case because of the simplicity of our setting)
  ```lean
  structure Header where
    binders     : Array (TSyntax ``Parser.Term.bracketedBinder)
    argNames    : Array Name
    targetNames : Array Name
    targetType  : Term
  ```

### Evaluate the functions

Continuing the steps from above: 

  3) Plug the functions from step (2) into an instance generating command. This is also done for us using the `mkInstanceCmds` utility:
  ```lean
  def mkInstanceCmds (ctx : Context) (className : Name) (typeNames : Array Name) (useAnonCtor := true) : TermElabM (Array Command)
  ```
  Looking at the source code for this function we see that it, in essence, returns an array of commands of the form
  ```lean
  `(instance $binders:implicitBinder* : $type := $val)
  ```
  where the `binders`, `type`, and `val` are all determined from the Context and the other inputs of the function (in particular, it uses the auxiliary function names from `mkContext`).

Putting all this together, you can find an example implementation of our instance handler [here](https://gist.github.com/mpenciak/fa3291df4496ed9455ad72b8a50873b8). Each step is separated out with self-explanatory names and type signatures, but we explain it nonetheless:

`mkMyNameHandler` is the handler which we initialize at the bottom of the file using `registerBuiltinDerivingHandler`. It relies on `mkMyNameInstanceCmds` to return the commands to elaborate to add the instance to the environment via `elabCommand`.

`mkMyNameInstanceCmds` generates the context, and returns the array of commands to run. These are split into the function declarations in `mkMyNameFuns`, and the instance declarations from the ` mkInstanceCmds` utility.

Finally, `mkMyNameFuns` is the actual place where we apply some of what we learned about metaprogramming and write down the actual function that defines the instance:

```lean
`(private def $(mkIdent auxFunName):ident $header.binders:bracketedBinder* : String := $(Syntax.mkStrLit name.toString))
```

In order for `mkInstanceCmds` to refer to the right function when generating the instance declaration, the identifier for the function, as we explained above, must come from the `auxFunNames` generated in the `Context`. The `mkHeader` command does all the work for us in defining the binders for the function to match the appropriate type signature of `MyName.nameID`, and finally the `rhs` is where we define the desired behavior of the function. In this case, it should just return the string literal of the name of the inductive.

And with that (in a separate file, with the right imports!) we can derive our instances automatically:

```lean
inductive test 
  | unit
  | recr : test → test
deriving MyName

structure foo :=
  bar : Nat
  baz : Bool
deriving MyName

open test in
#eval MyName.nameID unit -- "test"
#eval MyName.nameID (⟨3, true⟩ : foo) -- "foo"
```

### Some final notes

Because our typeclass was so simple, we didn't run into a lot of the complications that come up while trying to derive more involved instances. A more complicated example is defined [here](https://github.com/yatima-inc/Serde.lean/blob/mp/serialize-handler/Serde/Deriving/Serialize.lean) for a Serde serializer I wrote (which is extremely close to the `FromToJson` serializer built into Lean). In this section I'll list some more utilities and patterns that can be useful in other serializers:

* If the functionality of your instance handler should change on whether or not it's being derived for an `inductive` or a `structure`, then there's a handy 
  ```lean
  def Lean.isStructure (env : Environment) (constName : Name) : Bool
  ```` 
* On the topic of structures, you will also likely want the names of the fields:
```lean
def getStructureFieldsFlattened (env : Environment) (structName : Name) (includeSubobjectFields := true) : Array Name
```
* For most applications, we will likely want our typeclass to behave differently on an inductive constructed via different constructors. The way we would implement this "by hand" is with a match statement, so that is exactly how we'll implement it using metaprogramming. In the Serde (and `FromToJson`) serializers we split this up into a number of steps
  1) Make the match statement: 
    ```lean
     `(match $[$discrs],* with $altsArray:matchAlt*)
    ```
    here the discriminants are generated via 
    ```lean
    def mkDiscrs (header : Header) (indVal : InductiveVal) : TermElabM (Array (TSyntax ``Parser.Term.matchDiscr))
    ```
    utility. This utility needs an `InductiveVal` which we can generate using `getConstInfoInduct [MonadEnv m] : Name -> m InductiveVal` on the name of our inductive. The ``` `altsArray : `matchAlt ``` can be constructed by looping over all the constructors, which are conveniently packaged in `InductiveVal.ctors : InductiveVal -> List Name`.
  2) Make a single match's LHS by collecting all of the binders using `indVal.numIndices` and `indVal.numParams` and `ctorInfo.numFields`. Here `ctorInfo : ConstructorVal`, which you can find using `getConstInfoCtor`. For each field in the constructor, it is wise to make a fresh name and pass it to the next step to implement the functionality.
  3) Write the RHS for each constructor to implement the desired behavior. This part of the metaprogramming exercise is as complicated as the behavior you're trying to derive.
    
There's a ton I still have left to learn about deriving typeclass handlers, but hopefully this helps you write some deriving handlers yourself!

## Addendum: A journey through Lean 4's source code

The keywords `deriving _` is actually a language extension written in Lean itself. In this section we'll look at how it's implemented, and get a sense of why doing the above is enough to generate an instance automatically. This section is totally optional, and may be more confusing than anything else. But it helped me figure out what was going on when I learned the subject.

### The Parser

If we want to know what's happening when we write `deriving MyName`, we should start from the beginning on what Lean sees when parsing `deriving MyName`. This comes from reading through the [Lean command parser](https://github.com/leanprover/lean4/blob/master/src/Lean/Parser/Command.lean), for which the key component is included below (this is most clearly read from the bottom to the top):

```lean
def ctor             := leading_parser "\n| " >> ppIndent (declModifiers true >> ident >> optDeclSig)
def derivingClasses  := sepBy1 (group (ident >> optional (" with " >> Term.structInst))) ", "
def optDeriving      := leading_parser optional (ppLine >> atomic ("deriving " >> notSymbol "instance") >> derivingClasses)
def «inductive»      := leading_parser "inductive " >> declId >> optDeclSig >> optional (symbol " :=" <|> " where") >> many ctor >> optDeriving
```

On an inductive declaration command, Lean expects to see the keyword `inductive` followed by a declaration ID, an optional signature, optional `:=` or `where`, a bunch of constructors, and finally an "optional deriving token".

`optDriving` is then a parser that looks for the keyword `deriving` (but not followed immediately by `inductive`!) and a list of declarations we intend on deriving the instances for. This is the work of the `derivingClasses` parser. 

Once this is parsed, where is the command of declaring the inductive elaborated? After a little digging around the Lean 4 source code, a reasonable guess would be in the `elabDeclaration` found [here](https://github.com/leanprover/lean4/blob/master/src/Lean/Elab/Declaration.lean). The type of `elabDeclaration` is `Syntax -> CommandM Unit`, and reading the source we can roughly see that if the ```declKind == ``Lean.Parser.Command.«inductive»``` we should evaluate `elabClassInductive` on the declaration. We can keep on going down the rabbit hole by `ctrl`-clicking on functions that look interesting (this is actually kind of fun, I learned a bunch of things about Lean's inner workings... For example, did you know you can write `stx[n]` to access parts of a `stx : Lean.Syntax`?)

### The elaboration

We have to first convert the `Syntax` object into an `InductiveView` the definition of which can be found [here](https://github.com/leanprover/lean4/blob/master/src/Lean/Elab/Inductive.lean). One particular part of an `InductiveView` is an array of `DerivingClassView`s. 

```lean
structure DerivingClassView where
  ref : Syntax
  className : Name
  args? : Option (TSyntax ``Parser.Term.structInst)
```

The next elaborated function, `elabInductiveViews`, in particular calls  `applyDerivingHandlers`... Now we're on to something! The type of this function is `InductiveView -> CommandElabM Unit`, so presumably we should expect the effect of this computation should be a side-effect on the Environment!

Parsing `applyDerivingHandlers` we see a nested loop over all classes being derived and all inductive views wherein the `DerivingClassView.applyHandler` function is applied to all relevant `DerivingClassView`s. It turns out this takes us to the [`Lean.Elab.Deriving.Basic`](https://github.com/leanprover/lean4/blob/master/src/Lean/Elab/Deriving/Basic.lean) file (which is probably where we should have started to begin with!)

In fact`.applyHandler` is the last function in the file, and as is usual in Lean files sometimes it is easiest to parse their contents reading bottom to top. Doing so, we see that in the definition of `.applyHandler`: the function `applyDerivingHandlers` is called. `applyDerivingHandlers` now looks for `derivingHandlersRef` which is an `IO.Ref (NameMap DerivingHandler)`. 

```lean
def applyDerivingHandlers (className : Name) (typeNames : Array Name) (args? : Option (TSyntax ``Parser.Term.structInst)) : CommandElabM Unit := do
  match (← derivingHandlersRef.get).find? className with
  | some handler =>
    unless (← handler typeNames args?) do
      defaultHandler className typeNames
  | none => defaultHandler className typeNames

...

def DerivingClassView.applyHandlers (view : DerivingClassView) (declNames : Array Name) : CommandElabM Unit :=
  withRef view.ref do applyDerivingHandlers view.className declNames view.args?
```

(Along the way we also see `elabDeriving` which is the elaboration of some new syntax we were protecting against earlier: ``` `(deriving instance $[$classes $[with $argss?]?],* for $[$declNames],*)```)

Parsing this function, we see that given `className` (the name of the class we intend on deriving) and `typeNames` (the names of the inductives and structures we want to derive the class for), the function `applyDerivingHandlers` looks for the classname in `derivingHandlersRef` (more on this later), and if it finds some `handler : DerivingHandler)`, the function `applyDerivingHandlers` evaluates the handler and expects a boolean.

### Handlers


A `DerivingHandler` is a type defined as functions:

```lean
def DerivingHandler := (typeNames : Array Name) → (args? : Option (TSyntax ``Parser.Term.structInst)) → CommandElabM Bool
```
In essence we should expect the `DerivingHandler` to make some modifications to the environment (for example, evaluating a command that adds a particular typeclass instance) and return a boolean on whether it is 

Back to the main thread: `applyDerivingHandlers` evaluates the handler and expects a boolean. When `false`, this boolean signals to Lean that the deriving failed, and that some default handler should be applied instead.

So where is Lean looking for these `DerivingHandler`s? We saw it being looked for in the reference `derivingHandlerRef`. We see its initialization:

```lean
builtin_initialize derivingHandlersRef : IO.Ref (NameMap DerivingHandler) ← IO.mkRef {}
```

which we can interpret roughly as there being an ambient `Std.RBMap` from `Name`s to `DerivingHandler`s. That must be what we want! We just need to find a way to modify this particular `IO.Ref`. 

This is done by the function right below: `registerBuiltinDerivingHandlerWithArgs` (or the alternative without arguments) which simply (after a few checks) evaluates the function `fun (m : NameMap DerivingHandler) => m.insert className handler` to the state.

We can now rest reassured that the path we took in the last section is exactly that what was necessary given the infrastructure in place.