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


### 
What is a block? A block is a lambda expression that captures (or closes over) its environment at creation-time. We will see later what it means exactly. For now, imagine a block as an anonymous function or method. A block is a piece of code whose execution is frozen and can be kicked in using messages. Blocks are defined by square brackets.

If you execute and print the result of the following code, you will not get 3, but a block. Indeed, you did not ask for the block value, but just for the block itself, and you got it.

```pharo&caption=Block definition
[ 1 + 2 ] 
	--> [ 1 + 2 ]
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
>[ 1 + 2 ] cull: 5 cull: 6 
3
>[ :x | 2 + x ] cull: 5
7
>[ :x | 2 + x ] cull: 5 cull: 3
7
>[ :x :y | 1 + x + y ] cull: 5 cull: 2
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
- `on: exception fork: catchBlock`. Executes the receiver. If an exception `exception` is raised, fork a new process, which will handle the error. The original process will continue running as if the receiver evaluation finished and answered nil, i.e. an expression like: `[ self error: 'some error' ] on: Error fork: [ :ex | 123 ]` will always answer nil to the original process. The context stack, starting from the context which sent this message to the receiver and up to the top of the stack will be transferred to the forked process, with the catch block on top. 
Eventually, the catch block will be evaluated in the forked process.

Some messages are related to process scheduling. We list the most important ones. Since this Chapter is not about concurrent programming in Pharo, we will not go deep into them.

- `fork`. Create and schedule a `Process` evaluating the receiver.
- `forkAt: aPriority`. Create and schedule a `Process` evaluating the receiver at the given priority. Answer the newly created process.
- `newProcess`. Answer a Process evaluating the receiver. The process is not scheduled.




