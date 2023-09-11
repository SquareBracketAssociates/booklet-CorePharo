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

This chapter is the companion one of the one on exceptions. 
In the Pharo by Example book, we presented how to write and use blocks. 
On the contrary, this chapter focuses on deep aspects and their runtime behavior.