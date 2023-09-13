## Understanding Blocks

Lexically-scoped block closures, blocks, in short, are a powerful and essential feature of Pharo. 
Without them, it would be difficult to have such a small and compact syntax. 
The use of blocks is key to getting conditionals and loops as library messages and not hardcoded in the language syntax. 
This is why we can say that blocks work extremely well with the message-passing syntax of Pharo.

In addition, blocks are effective in improving the readability, reusability, and efficiency of code.  The fine dynamic runtime semantics of blocks, however, is not well documented. For example, blocks in the presence of return statements behave like an escaping mechanism and while this can lead to ugly code when used to its extreme, it is important to understand it. 

In this chapter, you will learn about the central notion of context (objects that represent points in program execution)  and the capture of variables at block creation time. 
You will learn how block returns can change program flow. 
Finally, to understand blocks, we describe how programs execute and in particular, we present contexts, also called activation records, which represent a given execution state. 
We will show how contexts are used during the block execution.

This chapter is the companion chapter of the one on exceptions. 
In the Pharo by Example book, we presented how to write and use blocks. 
On the contrary, this chapter focuses on deep aspects and their runtime behavior.


### What is a block? 

A block is a lambda expression that captures (or closes over) its environment at creation-time. We will see later what it means exactly. For now, imagine a block as an anonymous function or method. A block is a piece of code whose execution is frozen and can be kicked in using messages. Blocks are defined by square brackets.

If you execute and print the result of the following code, you will not get 3, but a block. Indeed, you did not ask for the block value, but just for the block itself, and you got it.

```pharo&caption=Block definition
> [ 1 + 2 ] 
[ 1 + 2 ]
```

 A block is executed by sending the `value` message to it. 
 More precisely, blocks can be executed using 
 - `value` (when no argument is mandatory), 
 - `value:` (when the block requires one argument), 
 - `value:value:` (for two arguments), 
 - `value:value:value:` (for three) and `valueWithArguments: anArray` (for more arguments). 
 
 These messages are the basic and historical API for block execution. They are presented in the Pharo by Example book.

```pharo&caption=Block execution
> [ 1 + 2 ] value 
3
```

```pharo&caption=Block execution
> [ :x | x + 2 ] value: 5 
7
```


### Some handy extensions

Beyond the `value` messages, Pharo includes some handy messages
such as `cull:` and friends to support the evaluation of blocks even
in the presence of more values than necessary. 

`cull:` will raise
an error if the receiver requires more arguments than provided. The
`valueWithPossibleArgs:` message is similar to `cull:` but takes
an array of parameters to pass to a block as argument. 
If the block requires more arguments than provided, `valueWithPossibleArgs:`
will fill them with `nil`.

```Cull: examples
> [ 1 + 2 ] cull: 5 
3
> [ 1 + 2 ] cull: 5 cull: 6 
3
> [ :x | 2 + x ] cull: 5
7
> [ :x | 2 + x ] cull: 5 cull: 3
7
> [ :x :y | 1 + x + y ] cull: 5 cull: 2
8
```

As a general remark do not use `cull:` because it shows there is a kind of misuse
of blocks. Using a real object instead is in the long run a better choice. 
Remember that `cull:` is using reflective operations to execute.

```
> [ :x :y | 1 + x + y ] cull: 5
error because the block needs 2 arguments.
> [ :x :y | 1 + x + y ] valueWithPossibleArgs: #(5)
error because 'y' is nil and '+' does not accept nil as a parameter.
```



### Other messages

Some messages are useful for profiling execution. 
We list three: 

- `bench`. Returns how many times the receiver block can be evaluated in 5 seconds.
- `durationToRun`. Answers the duration (instance of class `Duration`) taken to evaluate the receiver block.
- `timeToRun`. Answers the number of milliseconds taken to evaluate this block.



Some messages are related to error handling (as explained in the companion chapter on exception).

- `ensure: terminationBlock`. Executes the termination block after executing the receiver, regardless of whether the receiver's execution completes.
- `ifCurtailed: onErrorBlock`. Executes the receiver, and, if the execution does not complete, executes the error block. If receiver execution finishes normally, the error block is not executed
- `on: exception do: catchBlock`. Executes the receiver. If an exception `exception` is raised, executes the catch block.
- `on: exception fork: catchBlock`. Executes the receiver. If an exception `exception` is raised, fork a new process, which will handle the error. The original process will continue running as if the receiver evaluation finished and answered nil, i.e. an expression like: `[ self error: 'some error' ] on: Error fork: [ :ex | 123 ]` will always answer nil to the original process. The context stack, starting from the context which sent this message to the receiver and up to the top of the stack will be transferred to the forked process, with the catch block on top.  Eventually, the catch block will be evaluated in the forked process.

Some messages are related to process scheduling. We list the most important ones. Since this Chapter is not about concurrent programming in Pharo, we will not go deep into them.

- `fork`. Create and schedule a `Process` evaluating the receiver.
- `forkAt: aPriority`. Create and schedule a `Process` evaluating the receiver at the given priority. Answer the newly created process.
- `newProcess`. Answer a Process evaluating the receiver. The process is not scheduled.



### Variables and blocks

A block can have its own temporary variables. Such variables are initialized during each block execution and are local to the block. We will see later how such variables are kept. Now the question we want to make clear is what happens when a block refers to other (non-local) variables. A block will close over the external variables it uses. It means that even if the block is executed later in an environment that does not lexically contain the variables used by a block, the block will still have access to the variables during its execution. Later, we will present how local variables are implemented and stored using contexts.

In Pharo, private variables (such as `self`, instance variables, method temporaries, and arguments) are lexically scoped: an expression in a method can access the variables visible from that method, but the same expression put in another method or class cannot access the same variables because they are not in the scope of the expression (i.e. visible from the expression). 

At runtime, the variables that a block can access, are bound (get a value associated with them) in _the context_ in which the block that contains them is _defined_, rather than the context in which the block is evaluated. It means that a block, when evaluated somewhere else can access variables that were in its scope (visible to the block) when the block was _created_. Traditionally, the context in which a block is defined is named the _block home context_.

The block home context represents a particular point of execution (since this is a program execution that created the block in the first place), therefore this notion of block home context is represented by an object that represents program execution: a _context_ object in Pharo. In essence, a context (called stack frame or activation record in other languages) represents information about the current evaluation step such as the context from which the current one is executed, the next bytecode to be executed, and temporary variable values. A context is a Pharo execution stack element. This is important and we will come back later to this concept.

**A block is created inside a context (an object that represents a point in the execution).**

In the following sections we will experiment a bit to understand how variables are bound in a block. Define a class named `Bexp` (for `BlockExperiment`):

```language=pharo
Object << #Bexp
	package: 'BlockExperiment'
```


### Experiment 1: Variable lookup

A variable is looked up in the block definition context. We define two methods: one that defines a variable `t` and sets it to 42 and a block `[t traceCr]` and one that defines a new variable with the same name and executes a block defined elsewhere.



```language=pharo
Bexp >> setVariableAndDefineBlock
	| t |
	t := 42.
	self evaluateBlock: [ t traceCr ]
	
Bexp >> evaluateBlock: aBlock
	| t |
	t := nil.
	aBlock value	
```
```
> Bexp new setVariableAndDefineBlock 
42
```


Executing the `Bexp new setVariableAndDefineBlock` expression prints 42 in the Transcript (message `traceCr`). The value of the temporary variable `t` defined in the `setVariableAndDefineBlock` method is the one used rather than the one defined inside the method `evaluateBlock:` even if the block is evaluated during the execution of this method. The variable `t` is  looked up in the context of the block creation (context created during the execution of the method `setVariableAndDefineBlock` and not in the context of the block evaluation 
(method `evaluateBlock:`).


Let's look at it in detail. Figure *@fig:variable@* shows the execution of the expression `Bexp new setVariableAndDefineBlock`. 

-  During the execution of method `setVariableAndDefineBlock`, a variable `t` is defined and it is assigned 42. Then a block is created and this block refers to the method activation context - which holds temporary variables (Step 1). 

-  The method `evaluateBlock:` defines its own local variable `t` with the same name as the one in the block. This is not this variable, however, that is used when the block is evaluated. While executing the method `evaluateBlock:` the block is evaluated (Step 2), during the execution of the expression `t traceCr` the non-local variable `t` is looked up in the _home context_ of the block i.e. the method context that _created_ the block and not the context of the currently executed method.

![Non-local variables are looked up in the method activation context where the block was _created_ and not where it is _evaluated_.](figures/variable.pdf  label=fig:variable)



**Non-local variables are looked up in the _home context_ of the block (i.e. the method context that _created_ the block) and not the context executing the block.**




### Experiment 2: Changing a variable value
Let's continue our experiments. The method `setVariableAndDefineBlock2` shows that a non-local variable value can be changed during the evaluation of a block. Executing `Bexp new setVariableAndDefineBlock2` prints 33, since 33 is the last value of the variable `t`.


```language=pharo
Bexp >> setVariableAndDefineBlock2
	| t |
	t := 42.
	self evaluateBlock: [ t := 33. t traceCr ]

> Bexp new setVariableAndDefineBlock2	
33
```


### Experiment 3: Accessing a shared non-local variable
Two blocks can share a non-local variable and they can modify the value of this variable at different moments. To see this, let us define a new method `setVariableAndDefineBlock3` as follows:

```language=pharo
Bexp >> setVariableAndDefineBlock3
	| t |
	t := 42.
	self evaluateBlock: [ t traceCr. t := 33. t traceCr ].
	self evaluateBlock: [ t traceCr. t := 66. t traceCr ].
	self evaluateBlock: [ t traceCr ]
```

```language=pharo
> Bexp new setVariableAndDefineBlock3
42
33
33
66 
66
```

`Bexp new setVariableAndDefineBlock3` will print 42, 33, 33, 66 and 66.
Here the two blocks `[ t := 33. t traceCr ]` and `[ t := 66. t traceCr ]` access the same variable `t` and can modify it. During the first execution of the method `evaluateBlock:` its current value `42` is printed, then the value is changed and printed. A similar situation occurs with the second call. 

This example shows that blocks share the location where variables are stored and also that a block does not copy the value of a captured variable. It just refers to the location of the variables and several blocks can refer to the same location.

### Experiment 4: Variable lookup is done at execution time

The following example shows that the value of the variable is looked up at runtime and not copied during the block creation. First add the instance variable `block` to the class `Bexp`.

```language=pharo
Object << #Bexp
	slots: { #block };
	package: 'BlockExperiment'
```

Here the initial value of the variable `t` is 42. The block is created and stored in the instance variable `block` but the value to `t` is changed to 69 before the block is evaluated. And this is the last value (69) that is effectively printed because it is looked up at execution time. Executing `Bexp new setVariableAndDefineBlock4` prints 69.



```language=pharo
Bexp >> setVariableAndDefineBlock4
	| t |
	t := 42.
	block := [ t traceCr: t ].
	t := 69.
	self evaluateBlock: block

> Bexp new setVariableAndDefineBlock4 
69
```



### Experiment 5: For method arguments.

We can expect that method arguments are bound in the context of the defining method. 
Let's illustrate this point now. Define the following methods.

```language=pharo
Bexp>>testArg
	self testArg: 'foo'.

Bexp>>testArg: arg
	block := [ arg traceCr ].
	self evaluateBlockAndIgnoreArgument: 'zork'.

Bexp>>evaluateBlockAndIgnoreArgument: arg
	block value.
```

Now executing `Bexp new testArg: 'foo'` prints `'foo'` even if in the method `evaluateBlockAndIgnoreArgument:` the temporary `arg` is redefined. In fact, each method invocation has its own values for the arguments.

### Experiment 6: self binding 

Now we may wonder if `self` is also captured.
To test we need another class. Let's simply define a new class and a couple of methods.
Add the instance variable `x` to the class `Bexp` and define the `initialize` method as follows:

```language=pharo
Object << #Bexp
	instanceVariableNames: 'block x'
	package: 'BlockExperiment'
```
```
Bexp >> initialize
	super initialize.
	x := 123.
```

Define another class named `Bexp2`.

```language=pharo
Object << #Bexp2
	slots: { #x };
	package: 'BlockExperiment'
```
```
Bexp2 >> initialize
	super initialize.
	x := 69.

Bexp2 >> evaluateBlock: aBlock
	aBlock value
```

Then define the methods in class `Bexp` that will invoke methods defined in `Bexp2`.

```language=pharo
Bexp >> evaluateBlock: aBlock
	Bexp2 new evaluateBlock: aBlock

Bexp >> evaluateBlock
	self evaluateBlock: [ self crTrace ; traceCr: x ]
```

```language=pharo
> Bexp new evaluateBlock
a Bexp123  "and not a Bexp269"
```

Now when we execute `Bexp new evaluateBlock`, we get `a Bexp123` printed in the Transcript 
showing that a block captures `self` too since an instance of `Bexp2` executed the block but the 
printed object (`self`) is the original `Bexp` instance that was accessible at the block creation time.


We showed that blocks capture variables that are reached from the context in which the block was defined and not where they are executed. Blocks keep references to variable locations that can be shared between multiple blocks. 



% ============================================================================
### Block-local variables
% ============================================================================

As we saw previously a block is a lexical closure that is connected to the place where it is defined. In the following, we will illustrate this connection by showing that block local variables are allocated in the execution context linked to their creation. 
We will show the difference when a variable is local to a block or to a method (see Figure *@blockLocalTempExecution@*).

##### Block allocation.

 Implement the following method `blockLocalTemp`.

```language=pharo
Bexp >> blockLocalTemp
	| collection |
	collection := OrderedCollection new.
	#(1 2 3) do: [ :index |
		| temp |
		temp := index.
		collection add: [ temp ] ].
	^ collection collect: [ :each | each value ]
```

Let's comment `blockLocalTemp` definition: we create a loop that stores the current index (a block argument) in a temporary variable `temp` created in the loop. We then store a block that accesses this variable in a collection. After the loop, we execute each accessing block and return the collection of values. 

```
> Bexp new blockLocalTemp asArray
#(1 2 3)
```

When we execute this method, we get a collection with 1, 2, and 3. This result shows that each block in the collection refers to a different `temp` variable.
This is due to the fact that an execution context is created for _each_ block creation (at each loop step) and that the block `[ temp ]` refers to this context.

![`blockLocalTemp` execution](figures/blockClosureLocalTemp label=blockLocalTempExecution&width=70)



##### Method allocation.

Now let us create a new method, `blockOutsideTemp`, that is the same as `blockLocalTemp` except that the variable `temp` is a method temporary variable instead of a block variable.

```language=pharo
Bexp >> blockOutsideTemp
	| collection temp |
	collection := OrderedCollection new.
	#(1 2 3) do: [ :index | 
		temp := index.
		collection add: [ temp ] ].
	^ collection collect: [ :each | each value ]
```

When we execute the method `blockOutsideTemp`, we now get a collection with 3, 3, and 3. 

```
> Bexp new blockOutsideTemp asArray
#(3 3 3)
```

This result shows that each block in the collection now refers to a single variable `temp` allocated in the `blockOutsideTemp` context leading to the fact that `temp` is shared by the blocks.


![`blockClosureOutsideTemp` execution](figures/blockClosureOutsideTemp label=blockClosureOutsideTemp&width=70)



% ============================================================================
### Variables can outlive their defining method}
% ============================================================================

Non-block local variables referred to by a block continue to be accessible and shared with other expressions even if the method execution is terminated. We say that variables _outlive_ the method execution that defined them.
Let's look at some examples.

##### Method-Block Sharing.

 We start with a simple example showing that a variable is shared between a method and a block (as in the previous experiments in fact). Define the following method `foo` which defines a temporary variable `a`.

```language=pharo
Bexp >> foo
	| a |
	[ a := 0 ] value.
	^ a
```
```	
> Bexp new foo
0	
```

When we execute `Bexp new foo`, we get 0 and not nil.
What you see here is that the value is shared between the method body and the block.
 Inside the method body, we can access the variable whose value was set by the block evaluation.
Both the method and block bodies access the same temporary variable `a`.

Let's make it slightly more complicated. Define the method `twoBlockArray` as follows:

```language=pharo
Bexp >> twoBlockArray
	| a |
	a := 0.
	^ {[ a := 2] . [a]}
```

The method `twoBlockArray` defines a temporary variable `a`. It sets the value of `a`
to zero and returns an array whose first element is a block setting the value of `a` to 2 and second element is a block just returning the value of the temporary variable `a`.

Now we store the array returned by `twoBlockArray` and evaluate the blocks stored in the array. This is what the following code snippet is doing.

```language=pharo
| res |
res := Bexp new twoBlockArray.
res second value. 
	0
res first value.
res second value. 
	 2
```

We see that the first block execution modify the variable and that this variable is shared between the two blocks.
You can also define the code as follows and open a transcript to see the results.

```language=pharo
| res |
res := Bexp new twoBlockArray.
res second value traceCr.
res first value.
res second value traceCr.
```
Let us step back and look at an *important* point.

- In the previous code snippet when the expressions `res second value` and `res first value` are executed, the method `twoBlockArray` has _already finished its execution_ - as such it is not on the execution stack anymore. Still, the temporary variable `a` can be accessed and set to a new value. This experiment shows that the variables referred to by a block may live longer than the method that created the block that refers to them. We say that the variables _outlive the execution of their defining method_.

 You can see from this example that while temporary variables are somehow stored in an activation context, the implementation is a bit more subtle than that. The block implementation needs to keep referenced variables in a structure that is not in the execution stack but lives on the heap. The compiler performs some analysis and when it detects that a variable may outlive its creation context, it allocates the variables in a structure that is not allocated on the execution stack. 



