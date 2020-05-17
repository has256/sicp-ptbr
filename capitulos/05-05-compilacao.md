# 5.5 Compilation

The explicit-control evaluator of [5.4](5_002e4.xhtml#g_t5_002e4) is a register machine whose controller interprets Scheme programs. In this section we will see how to run Scheme programs on a register machine whose controller is not a Scheme interpreter.

The explicit-control evaluator machine is universal—it can carry out any computational process that can be described in Scheme. The evaluator’s controller orchestrates the use of its data paths to perform the desired computation. Thus, the evaluator’s data paths are universal: They are sufficient to perform any computation we desire, given an appropriate controller.^[This is a theoretical statement. We are not claiming that the evaluator’s data paths are a particularly convenient or efficient set of data paths for a general-purpose computer. For example, they are not very good for implementing high-performance floating-point calculations or calculations that intensively manipulate bit vectors.]

Commercial general-purpose computers are register machines organized around a collection of registers and operations that constitute an efficient and convenient universal set of data paths. The controller for a general-purpose machine is an interpreter for a register-machine language like the one we have been using. This language is called the *native language* of the machine, or simply *machine language*. Programs written in machine language are sequences of instructions that use the machine’s data paths. For example, the explicit-control evaluator’s instruction sequence can be thought of as a machine-language program for a general-purpose computer rather than as the controller for a specialized interpreter machine.

There are two common strategies for bridging the gap between higher-level languages and register-machine languages. The explicit-control evaluator illustrates the strategy of interpretation. An interpreter written in the native language of a machine configures the machine to execute programs written in a language (called the *source language*) that may differ from the native language of the machine performing the evaluation. The primitive procedures of the source language are implemented as a library of subroutines written in the native language of the given machine. A program to be interpreted (called the *source program*) is represented as a data structure. The interpreter traverses this data structure, analyzing the source program. As it does so, it simulates the intended behavior of the source program by calling appropriate primitive subroutines from the library.

In this section, we explore the alternative strategy of *compilation*. A compiler for a given source language and machine translates a source program into an equivalent program (called the *object program*) written in the machine’s native language. The compiler that we implement in this section translates programs written in Scheme into sequences of instructions to be executed using the explicit-control evaluator machine’s data paths.^[Actually, the machine that runs compiled code can be simpler than the interpreter machine, because we won’t use the `exp` and `unev` registers. The interpreter used these to hold pieces of unevaluated expressions. With the compiler, however, these expressions get built into the compiled code that the register machine will run. For the same reason, we don’t need the machine operations that deal with expression syntax. But compiled code will use a few additional machine operations (to represent compiled procedure objects) that didn’t appear in the explicit-control evaluator machine.]

Compared with interpretation, compilation can provide a great increase in the efficiency of program execution, as we will explain below in the overview of the compiler. On the other hand, an interpreter provides a more powerful environment for interactive program development and debugging, because the source program being executed is available at run time to be examined and modified. In addition, because the entire library of primitives is present, new programs can be constructed and added to the system during debugging.

In view of the complementary advantages of compilation and interpretation, modern program-development environments pursue a mixed strategy. Lisp interpreters are generally organized so that interpreted procedures and compiled procedures can call each other. This enables a programmer to compile those parts of a program that are assumed to be debugged, thus gaining the efficiency advantage of compilation, while retaining the interpretive mode of execution for those parts of the program that are in the flux of interactive development and debugging. In [5.5.7](#g_t5_002e5_002e7), after we have implemented the compiler, we will show how to interface it with our interpreter to produce an integrated interpreter-compiler development system.


### An overview of the compiler

Our compiler is much like our interpreter, both in its structure and in the function it performs. Accordingly, the mechanisms used by the compiler for analyzing expressions will be similar to those used by the interpreter. Moreover, to make it easy to interface compiled and interpreted code, we will design the compiler to generate code that obeys the same conventions of register usage as the interpreter: The environment will be kept in the `env` register, argument lists will be accumulated in `argl`, a procedure to be applied will be in `proc`, procedures will return their answers in `val`, and the location to which a procedure should return will be kept in `continue`. In general, the compiler translates a source program into an object program that performs essentially the same register operations as would the interpreter in evaluating the same source program.

This description suggests a strategy for implementing a rudimentary compiler: We traverse the expression in the same way the interpreter does. When we encounter a register instruction that the interpreter would perform in evaluating the expression, we do not execute the instruction but instead accumulate it into a sequence. The resulting sequence of instructions will be the object code. Observe the efficiency advantage of compilation over interpretation. Each time the interpreter evaluates an expression—for example, `(f 84 96)`—it performs the work of classifying the expression (discovering that this is a procedure application) and testing for the end of the operand list (discovering that there are two operands). With a compiler, the expression is analyzed only once, when the instruction sequence is generated at compile time. The object code produced by the compiler contains only the instructions that evaluate the operator and the two operands, assemble the argument list, and apply the procedure (in `proc`) to the arguments (in `argl`).

This is the same kind of optimization we implemented in the analyzing evaluator of [4.1.7](4_002e1.xhtml#g_t4_002e1_002e7). But there are further opportunities to gain efficiency in compiled code. As the interpreter runs, it follows a process that must be applicable to any expression in the language. In contrast, a given segment of compiled code is meant to execute some particular expression. This can make a big difference, for example in the use of the stack to save registers. When the interpreter evaluates an expression, it must be prepared for any contingency. Before evaluating a subexpression, the interpreter saves all registers that will be needed later, because the subexpression might require an arbitrary evaluation. A compiler, on the other hand, can exploit the structure of the particular expression it is processing to generate code that avoids unnecessary stack operations.

As a case in point, consider the combination `(f 84 96)`. Before the interpreter evaluates the operator of the combination, it prepares for this evaluation by saving the registers containing the operands and the environment, whose values will be needed later. The interpreter then evaluates the operator to obtain the result in `val`, restores the saved registers, and finally moves the result from `val` to `proc`. However, in the particular expression we are dealing with, the operator is the symbol `f`, whose evaluation is accomplished by the machine operation `lookup-variable-value`, which does not alter any registers. The compiler that we implement in this section will take advantage of this fact and generate code that evaluates the operator using the instruction

``` {.scheme}
(assign proc 
        (op lookup-variable-value)
        (const f)
        (reg env))
```

This code not only avoids the unnecessary saves and restores but also assigns the value of the lookup directly to `proc`, whereas the interpreter would obtain the result in `val` and then move this to `proc`.

A compiler can also optimize access to the environment. Having analyzed the code, the compiler can in many cases know in which frame a particular variable will be located and access that frame directly, rather than performing the `lookup-variable-value` search. We will discuss how to implement such variable access in [5.5.6](#g_t5_002e5_002e6). Until then, however, we will focus on the kind of register and stack optimizations described above. There are many other optimizations that can be performed by a compiler, such as coding primitive operations “in line” instead of using a general `apply` mechanism (see [Exercise 5.38](#Exercise-5_002e38)); but we will not emphasize these here. Our main goal in this section is to illustrate the compilation process in a simplified (but still interesting) context.


## 5.5.1 Structure of the Compiler

In [4.1.7](4_002e1.xhtml#g_t4_002e1_002e7) we modified our original metacircular interpreter to separate analysis from execution. We analyzed each expression to produce an execution procedure that took an environment as argument and performed the required operations. In our compiler, we will do essentially the same analysis. Instead of producing execution procedures, however, we will generate sequences of instructions to be run by our register machine.

The procedure `compile` is the top-level dispatch in the compiler. It corresponds to the `eval` procedure of [4.1.1](4_002e1.xhtml#g_t4_002e1_002e1), the `analyze` procedure of [4.1.7](4_002e1.xhtml#g_t4_002e1_002e7), and the `eval-dispatch` entry point of the explicit-control-evaluator in [5.4.1](5_002e4.xhtml#g_t5_002e4_002e1). The compiler, like the interpreters, uses the expression-syntax procedures defined in [4.1.2](4_002e1.xhtml#g_t4_002e1_002e2).^[Notice, however, that our compiler is a Scheme program, and the syntax procedures that it uses to manipulate expressions are the actual Scheme procedures used with the metacircular evaluator. For the explicit-control evaluator, in contrast, we assumed that equivalent syntax operations were available as operations for the register machine. (Of course, when we simulated the register machine in Scheme, we used the actual Scheme procedures in our register machine simulation.)] `Compile` performs a case analysis on the syntactic type of the expression to be compiled. For each type of expression, it dispatches to a specialized *code generator*:

``` {.scheme}
(define (compile exp target linkage)
  (cond ((self-evaluating? exp)
         (compile-self-evaluating 
          exp target linkage))
        ((quoted? exp) 
         (compile-quoted exp target linkage))
        ((variable? exp)
         (compile-variable 
          exp target linkage))
        ((assignment? exp)
         (compile-assignment
          exp target linkage))
        ((definition? exp)
         (compile-definition
          exp target linkage))
        ((if? exp)
         (compile-if exp target linkage))
        ((lambda? exp)
         (compile-lambda exp target linkage))
        ((begin? exp)
         (compile-sequence 
          (begin-actions exp) target linkage))
        ((cond? exp) 
         (compile 
          (cond->if exp) target linkage))
        ((application? exp)
         (compile-application 
          exp target linkage))
        (else
         (error "Unknown expression type: 
                 COMPILE" 
                exp))))
```


### Targets and linkages

`Compile` and the code generators that it calls take two arguments in addition to the expression to compile. There is a *target*, which specifies the register in which the compiled code is to return the value of the expression. There is also a *linkage descriptor*, which describes how the code resulting from the compilation of the expression should proceed when it has finished its execution. The linkage descriptor can require that the code do one of the following three things:

- continue at the next instruction in sequence (this is specified by the linkage descriptor `next`),
- return from the procedure being compiled (this is specified by the linkage descriptor `return`), or
- jump to a named entry point (this is specified by using the designated label as the linkage descriptor).

For example, compiling the expression `5` (which is self-evaluating) with a target of the `val` register and a linkage of `next` should produce the instruction

``` {.scheme}
(assign val (const 5))
```

Compiling the same expression with a linkage of `return` should produce the instructions

``` {.scheme}
(assign val (const 5))
(goto (reg continue))
```

In the first case, execution will continue with the next instruction in the sequence. In the second case, we will return from a procedure call. In both cases, the value of the expression will be placed into the target `val` register.


### Instruction sequences and stack usage

Each code generator returns an *instruction sequence* containing the object code it has generated for the expression. Code generation for a compound expression is accomplished by combining the output from simpler code generators for component expressions, just as evaluation of a compound expression is accomplished by evaluating the component expressions.

The simplest method for combining instruction sequences is a procedure called `append-instruction-sequences`. It takes as arguments any number of instruction sequences that are to be executed sequentially; it appends them and returns the combined sequence. That is, if $\langle\mspace{2mu} seq_{1}\rangle$ and $\langle\mspace{2mu} seq_{2}\rangle$ are sequences of instructions, then evaluating

``` {.scheme}
(append-instruction-sequences ⟨seq₁⟩ ⟨seq₂⟩)
```

produces the sequence

``` {.scheme}
⟨seq₁⟩
⟨seq₂⟩
```

Whenever registers might need to be saved, the compiler’s code generators use `preserving`, which is a more subtle method for combining instruction sequences. `Preserving` takes three arguments: a set of registers and two instruction sequences that are to be executed sequentially. It appends the sequences in such a way that the contents of each register in the set is preserved over the execution of the first sequence, if this is needed for the execution of the second sequence. That is, if the first sequence modifies the register and the second sequence actually needs the register’s original contents, then `preserving` wraps a `save` and a `restore` of the register around the first sequence before appending the sequences. Otherwise, `preserving` simply returns the appended instruction sequences. Thus, for example, `(preserving (list ⟨reg₁⟩ ⟨reg₂⟩) ⟨seg₁⟩ ⟨seg₂⟩)` produces one of the following four sequences of instructions, depending on how $\langle\mspace{2mu} seq_{1}\rangle$ and $\langle\mspace{2mu} seq_{2}\rangle$ use $\langle\mspace{2mu} reg_{1}\rangle$ and $\langle\mspace{2mu} reg_{2}\rangle$: 

$$\begin{array}{llll}
{\langle\mspace{2mu}{seq_{1}}\rangle} & \text{(save} & \text{(save} & {\text{(save}\mspace{41mu}\langle\mspace{2mu}{reg_{2}}\rangle\text{)}} \\
{\langle\mspace{2mu}{seq_{2}}\rangle} & {\mspace{41mu}\langle\mspace{2mu}{reg_{1}}\rangle\text{)}} & {\mspace{41mu}\langle\mspace{2mu}{reg_{2}}\rangle\text{)}} & {\text{(save}\mspace{41mu}\langle\mspace{2mu}{reg_{1}}\rangle\text{)}} \\
 & {\langle\mspace{2mu}{seq_{1}}\rangle} & {\langle\mspace{2mu}{seq_{1}}\rangle} & {\langle\mspace{2mu}{seq_{1}}\rangle} \\
 & \text{(restore} & \text{(restore} & {\text{(restore}\mspace{41mu}\langle\mspace{2mu}{reg_{1}}\rangle\text{)}} \\
 & {\mspace{41mu}\langle\mspace{2mu}{reg_{1}}\rangle\text{)}} & {\mspace{41mu}\langle\mspace{2mu}{reg_{2}}\rangle\text{)}} & {\text{(restore}\mspace{41mu}\langle\mspace{2mu}{reg_{2}}\rangle\text{)}} \\
 & {\langle\mspace{2mu}{seq_{2}}\rangle} & {\langle\mspace{2mu}{seq_{2}}\rangle} & {\langle\mspace{2mu}{seq_{2}}\rangle} \\
\end{array}$$

By using `preserving` to combine instruction sequences the compiler avoids unnecessary stack operations. This also isolates the details of whether or not to generate `save` and `restore` instructions within the `preserving` procedure, separating them from the concerns that arise in writing each of the individual code generators. In fact no `save` or `restore` instructions are explicitly produced by the code generators.

In principle, we could represent an instruction sequence simply as a list of instructions. `Append-instruction-sequences` could then combine instruction sequences by performing an ordinary list `append`. However, `preserving` would then be a complex operation, because it would have to analyze each instruction sequence to determine how the sequence uses its registers. `Preserving` would be inefficient as well as complex, because it would have to analyze each of its instruction sequence arguments, even though these sequences might themselves have been constructed by calls to `preserving`, in which case their parts would have already been analyzed. To avoid such repetitious analysis we will associate with each instruction sequence some information about its register use. When we construct a basic instruction sequence we will provide this information explicitly, and the procedures that combine instruction sequences will derive register-use information for the combined sequence from the information associated with the component sequences.

An instruction sequence will contain three pieces of information:

- the set of registers that must be initialized before the instructions in the sequence are executed (these registers are said to be *needed* by the sequence),
- the set of registers whose values are modified by the instructions in the sequence, and
- the actual instructions (also called *statements*) in the sequence.

We will represent an instruction sequence as a list of its three parts. The constructor for instruction sequences is thus

``` {.scheme}
(define (make-instruction-sequence 
         needs modifies statements)
  (list needs modifies statements))
```

For example, the two-instruction sequence that looks up the value of the variable `x` in the current environment, assigns the result to `val`, and then returns, requires registers `env` and `continue` to have been initialized, and modifies register `val`. This sequence would therefore be constructed as

``` {.scheme}
(make-instruction-sequence
 '(env continue)
 '(val)
 '((assign val
           (op lookup-variable-value)
           (const x)
           (reg env))
   (goto (reg continue))))
```

We sometimes need to construct an instruction sequence with no statements:

``` {.scheme}
(define (empty-instruction-sequence)
  (make-instruction-sequence '() '() '()))
```

The procedures for combining instruction sequences are shown in [5.5.4](#g_t5_002e5_002e4).

**Exercise 5.31:** In evaluating a procedure application, the explicit-control evaluator always saves and restores the `env` register around the evaluation of the operator, saves and restores `env` around the evaluation of each operand (except the final one), saves and restores `argl` around the evaluation of each operand, and saves and restores `proc` around the evaluation of the operand sequence. For each of the following combinations, say which of these `save` and `restore` operations are superfluous and thus could be eliminated by the compiler’s `preserving` mechanism:


``` {.scheme}
(f 'x 'y)
((f) 'x 'y)
(f (g 'x) y)
(f (g 'x) 'y)
```


**Exercise 5.32:** Using the `preserving` mechanism, the compiler will avoid saving and restoring `env` around the evaluation of the operator of a combination in the case where the operator is a symbol. We could also build such optimizations into the evaluator. Indeed, the explicit-control evaluator of [5.4](5_002e4.xhtml#g_t5_002e4) already performs a similar optimization, by treating combinations with no operands as a special case.

1.  Extend the explicit-control evaluator to recognize as a separate class of expressions combinations whose operator is a symbol, and to take advantage of this fact in evaluating such expressions.
2.  Alyssa P. Hacker suggests that by extending the evaluator to recognize more and more special cases we could incorporate all the compiler’s optimizations, and that this would eliminate the advantage of compilation altogether. What do you think of this idea?


## 5.5.2 Compiling Expressions

In this section and the next we implement the code generators to which the `compile` procedure dispatches.


### Compiling linkage code

In general, the output of each code generator will end with instructions—generated by the procedure `compile-linkage`—that implement the required linkage. If the linkage is `return` then we must generate the instruction `(goto (reg continue))`. This needs the `continue` register and does not modify any registers. If the linkage is `next`, then we needn’t include any additional instructions. Otherwise, the linkage is a label, and we generate a `goto` to that label, an instruction that does not need or modify any registers.^[This procedure uses a feature of Lisp called *backquote* (or *quasiquote*) that is handy for constructing lists. Preceding a list with a backquote symbol is much like quoting it, except that anything in the list that is flagged with a comma is evaluated.]

``` {.scheme}
(define (compile-linkage linkage)
  (cond ((eq? linkage 'return)
         (make-instruction-sequence 
          '(continue)
          '()
          '((goto (reg continue)))))
        ((eq? linkage 'next)
         (empty-instruction-sequence))
        (else
         (make-instruction-sequence '() '()
          `((goto (label ,linkage)))))))
```

The linkage code is appended to an instruction sequence by `preserving` the `continue` register, since a `return` linkage will require the `continue` register: If the given instruction sequence modifies `continue` and the linkage code needs it, `continue` will be saved and restored.

``` {.scheme}
(define (end-with-linkage 
         linkage instruction-sequence)
  (preserving '(continue)
   instruction-sequence
   (compile-linkage linkage)))
```


### Compiling simple expressions

The code generators for self-evaluating expressions, quotations, and variables construct instruction sequences that assign the required value to the target register and then proceed as specified by the linkage descriptor.

``` {.scheme}
(define (compile-self-evaluating 
         exp target linkage)
  (end-with-linkage
   linkage (make-instruction-sequence 
            '()
            (list target)
            `((assign ,target (const ,exp))))))

(define (compile-quoted exp target linkage)
  (end-with-linkage
   linkage
   (make-instruction-sequence
    '()
    (list target)
    `((assign 
       ,target
       (const ,(text-of-quotation exp)))))))

(define (compile-variable
         exp target linkage)
  (end-with-linkage 
   linkage
   (make-instruction-sequence 
    '(env)
    (list target)
    `((assign ,target
              (op lookup-variable-value)
              (const ,exp)
              (reg env))))))
```

All these assignment instructions modify the target register, and the one that looks up a variable needs the `env` register.

Assignments and definitions are handled much as they are in the interpreter. We recursively generate code that computes the value to be assigned to the variable, and append to it a two-instruction sequence that actually sets or defines the variable and assigns the value of the whole expression (the symbol `ok`) to the target register. The recursive compilation has target `val` and linkage `next` so that the code will put its result into `val` and continue with the code that is appended after it. The appending is done preserving `env`, since the environment is needed for setting or defining the variable and the code for the variable value could be the compilation of a complex expression that might modify the registers in arbitrary ways.

``` {.scheme}
(define (compile-assignment 
         exp target linkage)
  (let ((var (assignment-variable exp))
        (get-value-code
         (compile (assignment-value exp) 
                  'val
                  'next)))
    (end-with-linkage 
     linkage
     (preserving 
      '(env)
      get-value-code
      (make-instruction-sequence
       '(env val)
       (list target)
       `((perform (op set-variable-value!)
                  (const ,var)
                  (reg val)
                  (reg env))
         (assign ,target (const ok))))))))

(define (compile-definition 
         exp target linkage)
  (let ((var (definition-variable exp))
        (get-value-code
         (compile (definition-value exp)
                  'val
                  'next)))
    (end-with-linkage
     linkage
     (preserving 
      '(env)
      get-value-code
      (make-instruction-sequence
       '(env val)
       (list target)
       `((perform (op define-variable!)
                  (const ,var)
                  (reg val)
                  (reg env))
         (assign ,target (const ok))))))))
```

The appended two-instruction sequence requires `env` and `val` and modifies the target. Note that although we preserve `env` for this sequence, we do not preserve `val`, because the `get-value-code` is designed to explicitly place its result in `val` for use by this sequence. (In fact, if we did preserve `val`, we would have a bug, because this would cause the previous contents of `val` to be restored right after the `get-value-code` is run.)


### Compiling conditional expressions

The code for an `if` expression compiled with a given target and linkage has the form

``` {.scheme}
⟨compilation of predicate, 
 target val, linkage next⟩
 (test (op false?) (reg val))
 (branch (label false-branch))
true-branch
 ⟨compilation of consequent with given 
  target and given linkage or after-if⟩
false-branch
 ⟨compilation of alternative 
  with given target and linkage⟩
after-if
```

To generate this code, we compile the predicate, consequent, and alternative, and combine the resulting code with instructions to test the predicate result and with newly generated labels to mark the true and false branches and the end of the conditional.^[We can’t just use the labels `true-branch`, `false-branch`, and `after-if` as shown above, because there might be more than one `if` in the program. The compiler uses the procedure `make-label` to generate labels. `Make-label` takes a symbol as argument and returns a new symbol that begins with the given symbol. For example, successive calls to `(make-label 'a)` would return `a1`, `a2`, and so on. `Make-label` can be implemented similarly to the generation of unique variable names in the query language, as follows:] In this arrangement of code, we must branch around the true branch if the test is false. The only slight complication is in how the linkage for the true branch should be handled. If the linkage for the conditional is `return` or a label, then the true and false branches will both use this same linkage. If the linkage is `next`, the true branch ends with a jump around the code for the false branch to the label at the end of the conditional.

``` {.scheme}
(define (compile-if exp target linkage)
  (let ((t-branch (make-label 'true-branch))
        (f-branch (make-label 'false-branch))
        (after-if (make-label 'after-if)))
    (let ((consequent-linkage
           (if (eq? linkage 'next) 
               after-if
               linkage)))
      (let ((p-code 
             (compile (if-predicate exp)
                      'val
                      'next))
            (c-code
             (compile (if-consequent exp) 
                      target 
                      consequent-linkage))
            (a-code
             (compile (if-alternative exp)
                      target
                      linkage)))
        (preserving 
         '(env continue)
         p-code
         (append-instruction-sequences
          (make-instruction-sequence 
           '(val) 
           '()
           `((test (op false?) (reg val))
             (branch (label ,f-branch))))
          (parallel-instruction-sequences
           (append-instruction-sequences 
            t-branch c-code)
           (append-instruction-sequences
            f-branch a-code))
          after-if))))))
```

`Env` is preserved around the predicate code because it could be needed by the true and false branches, and `continue` is preserved because it could be needed by the linkage code in those branches. The code for the true and false branches (which are not executed sequentially) is appended using a special combiner `parallel-instruction-sequences` described in [5.5.4](#g_t5_002e5_002e4).

Note that `cond` is a derived expression, so all that the compiler needs to do handle it is to apply the `cond->if` transformer (from [4.1.2](4_002e1.xhtml#g_t4_002e1_002e2)) and compile the resulting `if` expression.


### Compiling sequences

The compilation of sequences (from procedure bodies or explicit `begin` expressions) parallels their evaluation. Each expression of the sequence is compiled—the last expression with the linkage specified for the sequence, and the other expressions with linkage `next` (to execute the rest of the sequence). The instruction sequences for the individual expressions are appended to form a single instruction sequence, such that `env` (needed for the rest of the sequence) and `continue` (possibly needed for the linkage at the end of the sequence) are preserved.

``` {.scheme}
(define (compile-sequence seq target linkage)
  (if (last-exp? seq)
      (compile (first-exp seq) target linkage)
      (preserving '(env continue)
       (compile (first-exp seq) target 'next)
       (compile-sequence (rest-exps seq)
                         target
                         linkage))))
```


### Compiling `lambda` expressions

`Lambda` expressions construct procedures. The object code for a `lambda` expression must have the form

``` {.scheme}
⟨construct procedure object 
 and assign it to target register⟩
⟨linkage⟩
```

When we compile the `lambda` expression, we also generate the code for the procedure body. Although the body won’t be executed at the time of procedure construction, it is convenient to insert it into the object code right after the code for the `lambda`. If the linkage for the `lambda` expression is a label or `return`, this is fine. But if the linkage is `next`, we will need to skip around the code for the procedure body by using a linkage that jumps to a label that is inserted after the body. The object code thus has the form

``` {.scheme}
⟨construct procedure object 
 and assign it to target register⟩
 ⟨code for given linkage⟩ or 
  (goto (label after-lambda))
 ⟨compilation of procedure body⟩
after-lambda
```

`Compile-lambda` generates the code for constructing the procedure object followed by the code for the procedure body. The procedure object will be constructed at run time by combining the current environment (the environment at the point of definition) with the entry point to the compiled procedure body (a newly generated label).^[e need machine operations to implement a data structure for representing compiled procedures, analogous to the structure for compound procedures described in [4.1.3](4_002e1.xhtml#g_t4_002e1_002e3):]

``` {.scheme}
(define (compile-lambda exp target linkage)
  (let ((proc-entry 
         (make-label 'entry))
        (after-lambda 
         (make-label 'after-lambda)))
    (let ((lambda-linkage
           (if (eq? linkage 'next)
               after-lambda
               linkage)))
      (append-instruction-sequences
       (tack-on-instruction-sequence
        (end-with-linkage 
         lambda-linkage
         (make-instruction-sequence 
          '(env)
          (list target)
          `((assign 
             ,target
             (op make-compiled-procedure)
             (label ,proc-entry)
             (reg env)))))
        (compile-lambda-body exp proc-entry))
       after-lambda))))
```

`Compile-lambda` uses the special combiner `tack-on-instruction-sequence` rather than `append-instruction-sequences` ([5.5.4](#g_t5_002e5_002e4)) to append the procedure body to the `lambda` expression code, because the body is not part of the sequence of instructions that will be executed when the combined sequence is entered; rather, it is in the sequence only because that was a convenient place to put it.

`Compile-lambda-body` constructs the code for the body of the procedure. This code begins with a label for the entry point. Next come instructions that will cause the run-time evaluation environment to switch to the correct environment for evaluating the procedure body—namely, the definition environment of the procedure, extended to include the bindings of the formal parameters to the arguments with which the procedure is called. After this comes the code for the sequence of expressions that makes up the procedure body. The sequence is compiled with linkage `return` and target `val` so that it will end by returning from the procedure with the procedure result in `val`.

``` {.scheme}
(define (compile-lambda-body exp proc-entry)
  (let ((formals (lambda-parameters exp)))
    (append-instruction-sequences
     (make-instruction-sequence 
      '(env proc argl)
      '(env)
      `(,proc-entry
        (assign env 
                (op compiled-procedure-env)
                (reg proc))
        (assign env
                (op extend-environment)
                (const ,formals)
                (reg argl)
                (reg env))))
     (compile-sequence (lambda-body exp)
                       'val
                       'return))))
```


## 5.5.3 Compiling Combinations

The essence of the compilation process is the compilation of procedure applications. The code for a combination compiled with a given target and linkage has the form

``` {.scheme}
⟨compilation of operator, 
 target proc, linkage next⟩
⟨evaluate operands and construct 
 argument list in argl⟩
⟨compilation of procedure call 
 with given target and linkage⟩
```

The registers `env`, `proc`, and `argl` may have to be saved and restored during evaluation of the operator and operands. Note that this is the only place in the compiler where a target other than `val` is specified.

The required code is generated by `compile-application`. This recursively compiles the operator, to produce code that puts the procedure to be applied into `proc`, and compiles the operands, to produce code that evaluates the individual operands of the application. The instruction sequences for the operands are combined (by `construct-arglist`) with code that constructs the list of arguments in `argl`, and the resulting argument-list code is combined with the procedure code and the code that performs the procedure call (produced by `compile-procedure-call`). In appending the code sequences, the `env` register must be preserved around the evaluation of the operator (since evaluating the operator might modify `env`, which will be needed to evaluate the operands), and the `proc` register must be preserved around the construction of the argument list (since evaluating the operands might modify `proc`, which will be needed for the actual procedure application). `Continue` must also be preserved throughout, since it is needed for the linkage in the procedure call.

``` {.scheme}
(define (compile-application 
         exp target linkage)
  (let ((proc-code 
         (compile (operator exp) 'proc 'next))
        (operand-codes
         (map (lambda (operand)
                (compile operand 'val 'next))
              (operands exp))))
    (preserving 
     '(env continue)
     proc-code
     (preserving 
      '(proc continue)
      (construct-arglist operand-codes)
      (compile-procedure-call 
       target
       linkage)))))
```

The code to construct the argument list will evaluate each operand into `val` and then `cons` that value onto the argument list being accumulated in `argl`. Since we `cons` the arguments onto `argl` in sequence, we must start with the last argument and end with the first, so that the arguments will appear in order from first to last in the resulting list. Rather than waste an instruction by initializing `argl` to the empty list to set up for this sequence of evaluations, we make the first code sequence construct the initial `argl`. The general form of the argument-list construction is thus as follows:

``` {.scheme}
⟨compilation of last operand, targeted to val⟩
(assign argl (op list) (reg val))
⟨compilation of next operand, targeted to val⟩
(assign argl (op cons) (reg val) (reg argl))
…
⟨compilation of first operand, targeted to val⟩
(assign argl (op cons) (reg val) (reg argl))
```

`Argl` must be preserved around each operand evaluation except the first (so that arguments accumulated so far won’t be lost), and `env` must be preserved around each operand evaluation except the last (for use by subsequent operand evaluations).

Compiling this argument code is a bit tricky, because of the special treatment of the first operand to be evaluated and the need to preserve `argl` and `env` in different places. The `construct-arglist` procedure takes as arguments the code that evaluates the individual operands. If there are no operands at all, it simply emits the instruction

``` {.scheme}
(assign argl (const ()))
```

Otherwise, `construct-arglist` creates code that initializes `argl` with the last argument, and appends code that evaluates the rest of the arguments and adjoins them to `argl` in succession. In order to process the arguments from last to first, we must reverse the list of operand code sequences from the order supplied by `compile-application`.

``` {.scheme}
(define (construct-arglist operand-codes)
  (let ((operand-codes 
         (reverse operand-codes)))
    (if (null? operand-codes)
        (make-instruction-sequence 
         '() 
         '(argl)
         '((assign argl (const ()))))
        (let ((code-to-get-last-arg
               (append-instruction-sequences
                (car operand-codes)
                (make-instruction-sequence 
                 '(val)
                 '(argl)
                 '((assign argl
                           (op list)
                           (reg val)))))))
          (if (null? (cdr operand-codes))
              code-to-get-last-arg
              (preserving 
               '(env)
               code-to-get-last-arg
               (code-to-get-rest-args
                (cdr operand-codes))))))))

(define (code-to-get-rest-args operand-codes)
  (let ((code-for-next-arg
         (preserving 
          '(argl)
          (car operand-codes)
          (make-instruction-sequence 
           '(val argl)
           '(argl)
           '((assign argl
                     (op cons)
                     (reg val)
                     (reg argl)))))))
    (if (null? (cdr operand-codes))
        code-for-next-arg
        (preserving 
         '(env)
         code-for-next-arg
         (code-to-get-rest-args 
          (cdr operand-codes))))))
```


### Applying procedures

After evaluating the elements of a combination, the compiled code must apply the procedure in `proc` to the arguments in `argl`. The code performs essentially the same dispatch as the `apply` procedure in the metacircular evaluator of [4.1.1](4_002e1.xhtml#g_t4_002e1_002e1) or the `apply-dispatch` entry point in the explicit-control evaluator of [5.4.1](5_002e4.xhtml#g_t5_002e4_002e1). It checks whether the procedure to be applied is a primitive procedure or a compiled procedure. For a primitive procedure, it uses `apply-primitive-procedure`; we will see shortly how it handles compiled procedures. The procedure-application code has the following form:

``` {.scheme}
(test (op primitive-procedure?) (reg proc))
 (branch (label primitive-branch))
compiled-branch
 ⟨code to apply compiled procedure 
  with given target and appropriate linkage⟩
primitive-branch
 (assign ⟨target⟩
         (op apply-primitive-procedure)
         (reg proc)
         (reg argl))
 ⟨linkage⟩
after-call
```

Observe that the compiled branch must skip around the primitive branch. Therefore, if the linkage for the original procedure call was `next`, the compound branch must use a linkage that jumps to a label that is inserted after the primitive branch. (This is similar to the linkage used for the true branch in `compile-if`.)

``` {.scheme}
(define (compile-procedure-call
         target linkage)
  (let ((primitive-branch 
         (make-label 'primitive-branch))
        (compiled-branch 
         (make-label 'compiled-branch))
        (after-call
         (make-label 'after-call)))
    (let ((compiled-linkage
           (if (eq? linkage 'next)
               after-call
               linkage)))
      (append-instruction-sequences
       (make-instruction-sequence 
        '(proc)
        '()
        `((test 
           (op primitive-procedure?)
           (reg proc))
          (branch 
           (label ,primitive-branch))))
       (parallel-instruction-sequences
        (append-instruction-sequences
         compiled-branch
         (compile-proc-appl 
          target
          compiled-linkage))
        (append-instruction-sequences
         primitive-branch
         (end-with-linkage
          linkage
          (make-instruction-sequence
           '(proc argl)
           (list target)
           `((assign 
              ,target
              (op apply-primitive-procedure)
              (reg proc)
              (reg argl)))))))
       after-call))))
```

The primitive and compound branches, like the true and false branches in `compile-if`, are appended using `parallel-instruction-sequences` rather than the ordinary `append-instruction-sequences`, because they will not be executed sequentially.


### Applying compiled procedures

The code that handles procedure application is the most subtle part of the compiler, even though the instruction sequences it generates are very short. A compiled procedure (as constructed by `compile-lambda`) has an entry point, which is a label that designates where the code for the procedure starts. The code at this entry point computes a result in `val` and returns by executing the instruction `(goto (reg continue))`. Thus, we might expect the code for a compiled-procedure application (to be generated by `compile-proc-appl`) with a given target and linkage to look like this if the linkage is a label

``` {.scheme}
(assign continue 
        (label proc-return))
 (assign val
         (op compiled-procedure-entry)
         (reg proc))
 (goto (reg val))
proc-return
 (assign ⟨target⟩ 
         (reg val))   ; included if target is not val
 (goto (label ⟨linkage⟩))   ; linkage code
```

or like this if the linkage is `return`.

``` {.scheme}
(save continue)
 (assign continue 
         (label proc-return))
 (assign val 
         (op compiled-procedure-entry)
         (reg proc))
 (goto (reg val))
proc-return
 (assign ⟨target⟩
         (reg val))   ; included if target is not val
 (restore continue)
 (goto (reg continue))   ; linkage code
```

This code sets up `continue` so that the procedure will return to a label `proc-return` and jumps to the procedure’s entry point. The code at `proc-return` transfers the procedure’s result from `val` to the target register (if necessary) and then jumps to the location specified by the linkage. (The linkage is always `return` or a label, because `compile-procedure-call` replaces a `next` linkage for the compound-procedure branch by an `after-call` label.)

In fact, if the target is not `val`, that is exactly the code our compiler will generate.^[Actually, we signal an error when the target is not `val` and the linkage is `return`, since the only place we request `return` linkages is in compiling procedures, and our convention is that procedures return their values in `val`.] Usually, however, the target is `val` (the only time the compiler specifies a different register is when targeting the evaluation of an operator to `proc`), so the procedure result is put directly into the target register and there is no need to return to a special location that copies it. Instead, we simplify the code by setting up `continue` so that the procedure will “return” directly to the place specified by the caller’s linkage:

``` {.scheme}
⟨set up continue for linkage⟩
(assign val 
        (op compiled-procedure-entry)
        (reg proc))
(goto (reg val))
```

If the linkage is a label, we set up `continue` so that the procedure will return to that label. (That is, the `(goto (reg continue))` the procedure ends with becomes equivalent to the `(goto (label ⟨linkage⟩))` at `proc-return` above.)

``` {.scheme}
(assign continue 
        (label ⟨linkage⟩))
(assign val
        (op compiled-procedure-entry)
        (reg proc))
(goto (reg val))
```

If the linkage is `return`, we don’t need to set up `continue` at all: It already holds the desired location. (That is, the `(goto (reg continue))` the procedure ends with goes directly to the place where the `(goto (reg continue))` at `proc-return` would have gone.)

``` {.scheme}
(assign val
        (op compiled-procedure-entry)
        (reg proc))
(goto (reg val))
```

With this implementation of the `return` linkage, the compiler generates tail-recursive code. Calling a procedure as the final step in a procedure body does a direct transfer, without saving any information on the stack.

Suppose instead that we had handled the case of a procedure call with a linkage of `return` and a target of `val` as shown above for a non-`val` target. This would destroy tail recursion. Our system would still give the same value for any expression. But each time we called a procedure, we would save `continue` and return after the call to undo the (useless) save. These extra saves would accumulate during a nest of procedure calls.^[Making a compiler generate tail-recursive code might seem like a straightforward idea. But most compilers for common languages, including C and Pascal, do not do this, and therefore these languages cannot represent iterative processes in terms of procedure call alone. The difficulty with tail recursion in these languages is that their implementations use the stack to store procedure arguments and local variables as well as return addresses. The Scheme implementations described in this book store arguments and variables in memory to be garbage-collected. The reason for using the stack for variables and arguments is that it avoids the need for garbage collection in languages that would not otherwise require it, and is generally believed to be more efficient. Sophisticated Lisp compilers can, in fact, use the stack for arguments without destroying tail recursion. (See [Hanson 1990](References.xhtml#Hanson-1990) for a description.) There is also some debate about whether stack allocation is actually more efficient than garbage collection in the first place, but the details seem to hinge on fine points of computer architecture. (See [Appel 1987](References.xhtml#Appel-1987) and [Miller and Rozas 1994](References.xhtml#Miller-and-Rozas-1994) for opposing views on this issue.)]

`Compile-proc-appl` generates the above procedure-application code by considering four cases, depending on whether the target for the call is `val` and whether the linkage is `return`. Observe that the instruction sequences are declared to modify all the registers, since executing the procedure body can change the registers in arbitrary ways.^[The variable `all-regs` is bound to the list of names of all the registers:] Also note that the code sequence for the case with target `val` and linkage `return` is declared to need `continue`: Even though `continue` is not explicitly used in the two-instruction sequence, we must be sure that `continue` will have the correct value when we enter the compiled procedure.

``` {.scheme}
(define (compile-proc-appl target linkage)
  (cond ((and (eq? target 'val)
              (not (eq? linkage 'return)))
         (make-instruction-sequence 
          '(proc)
          all-regs
          `((assign continue (label ,linkage))
            (assign 
             val 
             (op compiled-procedure-entry)
             (reg proc))
            (goto (reg val)))))
        ((and (not (eq? target 'val))
              (not (eq? linkage 'return)))
         (let ((proc-return 
                (make-label 'proc-return)))
           (make-instruction-sequence 
            '(proc)
            all-regs
            `((assign continue 
                      (label ,proc-return))
              (assign 
               val 
               (op compiled-procedure-entry)
               (reg proc))
              (goto (reg val))
              ,proc-return
              (assign ,target (reg val))
              (goto (label ,linkage))))))
        ((and (eq? target 'val)
              (eq? linkage 'return))
         (make-instruction-sequence 
          '(proc continue) 
          all-regs
          '((assign 
             val 
             (op compiled-procedure-entry)
             (reg proc))
            (goto (reg val)))))
        ((and (not (eq? target 'val))
              (eq? linkage 'return))
         (error "return linkage, 
                 target not val: COMPILE"
                target))))
```


## 5.5.4 Combining Instruction Sequences

This section describes the details on how instruction sequences are represented and combined. Recall from [5.5.1](#g_t5_002e5_002e1) that an instruction sequence is represented as a list of the registers needed, the registers modified, and the actual instructions. We will also consider a label (symbol) to be a degenerate case of an instruction sequence, which doesn’t need or modify any registers. So to determine the registers needed and modified by instruction sequences we use the selectors

``` {.scheme}
(define (registers-needed s)
  (if (symbol? s) '() (car s)))
(define (registers-modified s)
  (if (symbol? s) '() (cadr s)))
(define (statements s)
  (if (symbol? s) (list s) (caddr s)))
```

and to determine whether a given sequence needs or modifies a given register we use the predicates

``` {.scheme}
(define (needs-register? seq reg)
  (memq reg (registers-needed seq)))
(define (modifies-register? seq reg)
  (memq reg (registers-modified seq)))
```

In terms of these predicates and selectors, we can implement the various instruction sequence combiners used throughout the compiler.

The basic combiner is `append-instruction-sequences`. This takes as arguments an arbitrary number of instruction sequences that are to be executed sequentially and returns an instruction sequence whose statements are the statements of all the sequences appended together. The subtle point is to determine the registers that are needed and modified by the resulting sequence. It modifies those registers that are modified by any of the sequences; it needs those registers that must be initialized before the first sequence can be run (the registers needed by the first sequence), together with those registers needed by any of the other sequences that are not initialized (modified) by sequences preceding it.

The sequences are appended two at a time by `append-2-sequences`. This takes two instruction sequences `seq1` and `seq2` and returns the instruction sequence whose statements are the statements of `seq1` followed by the statements of `seq2`, whose modified registers are those registers that are modified by either `seq1` or `seq2`, and whose needed registers are the registers needed by `seq1` together with those registers needed by `seq2` that are not modified by `seq1`. (In terms of set operations, the new set of needed registers is the union of the set of registers needed by `seq1` with the set difference of the registers needed by `seq2` and the registers modified by `seq1`.) Thus, `append-instruction-sequences` is implemented as follows:

``` {.scheme}
(define (append-instruction-sequences . seqs)
  (define (append-2-sequences seq1 seq2)
    (make-instruction-sequence
     (list-union 
      (registers-needed seq1)
      (list-difference 
       (registers-needed seq2)
       (registers-modified seq1)))
     (list-union
      (registers-modified seq1)
      (registers-modified seq2))
     (append (statements seq1)
             (statements seq2))))
  (define (append-seq-list seqs)
    (if (null? seqs)
        (empty-instruction-sequence)
        (append-2-sequences 
         (car seqs)
         (append-seq-list (cdr seqs)))))
  (append-seq-list seqs))
```

This procedure uses some simple operations for manipulating sets represented as lists, similar to the (unordered) set representation described in [2.3.3](2_002e3.xhtml#g_t2_002e3_002e3):

``` {.scheme}
(define (list-union s1 s2)
  (cond ((null? s1) s2)
        ((memq (car s1) s2)
         (list-union (cdr s1) s2))
        (else
         (cons (car s1)
               (list-union (cdr s1) s2)))))

(define (list-difference s1 s2)
  (cond ((null? s1) '())
        ((memq (car s1) s2)
         (list-difference (cdr s1) s2))
        (else 
         (cons (car s1)
               (list-difference (cdr s1)
                                s2)))))
```

`Preserving`, the second major instruction sequence combiner, takes a list of registers `regs` and two instruction sequences `seq1` and `seq2` that are to be executed sequentially. It returns an instruction sequence whose statements are the statements of `seq1` followed by the statements of `seq2`, with appropriate `save` and `restore` instructions around `seq1` to protect the registers in `regs` that are modified by `seq1` but needed by `seq2`. To accomplish this, `preserving` first creates a sequence that has the required `save`s followed by the statements of `seq1` followed by the required `restore`s. This sequence needs the registers being saved and restored in addition to the registers needed by `seq1`, and modifies the registers modified by `seq1` except for the ones being saved and restored. This augmented sequence and `seq2` are then appended in the usual way. The following procedure implements this strategy recursively, walking down the list of registers to be preserved:^[Note that `preserving` calls `append` with three arguments. Though the definition of `append` shown in this book accepts only two arguments, Scheme standardly provides an `append` procedure that takes an arbitrary number of arguments.]

``` {.scheme}
(define (preserving regs seq1 seq2)
  (if (null? regs)
      (append-instruction-sequences seq1 seq2)
      (let ((first-reg (car regs)))
        (if (and 
             (needs-register? seq2 first-reg)
             (modifies-register? seq1 
                                 first-reg))
            (preserving 
             (cdr regs)
             (make-instruction-sequence
              (list-union 
               (list first-reg)
               (registers-needed seq1))
              (list-difference
               (registers-modified seq1)
               (list first-reg))
              (append `((save ,first-reg))
                      (statements seq1)
                      `((restore ,first-reg))))
             seq2)
            (preserving 
             (cdr regs)
             seq1
             seq2)))))
```

Another sequence combiner, `tack-on-instruction-sequence`, is used by `compile-lambda` to append a procedure body to another sequence. Because the procedure body is not “in line” to be executed as part of the combined sequence, its register use has no impact on the register use of the sequence in which it is embedded. We thus ignore the procedure body’s sets of needed and modified registers when we tack it onto the other sequence.

``` {.scheme}
(define (tack-on-instruction-sequence 
         seq body-seq)
  (make-instruction-sequence
   (registers-needed seq)
   (registers-modified seq)
   (append (statements seq)
           (statements body-seq))))
```

`Compile-if` and `compile-procedure-call` use a special combiner called `parallel-instruction-sequences` to append the two alternative branches that follow a test. The two branches will never be executed sequentially; for any particular evaluation of the test, one branch or the other will be entered. Because of this, the registers needed by the second branch are still needed by the combined sequence, even if these are modified by the first branch.

``` {.scheme}
(define (parallel-instruction-sequences 
         seq1 seq2)
  (make-instruction-sequence
   (list-union (registers-needed seq1)
               (registers-needed seq2))
   (list-union (registers-modified seq1)
               (registers-modified seq2))
   (append (statements seq1)
           (statements seq2))))
```


## 5.5.5 An Example of Compiled Code

Now that we have seen all the elements of the compiler, let us examine an example of compiled code to see how things fit together. We will compile the definition of a recursive `factorial` procedure by calling `compile`:

``` {.scheme}
(compile
 '(define (factorial n)
    (if (= n 1)
        1
        (* (factorial (- n 1)) n)))
 'val
 'next)
```

We have specified that the value of the `define` expression should be placed in the `val` register. We don’t care what the compiled code does after executing the `define`, so our choice of `next` as the linkage descriptor is arbitrary.

`Compile` determines that the expression is a definition, so it calls `compile-definition` to compile code to compute the value to be assigned (targeted to `val`), followed by code to install the definition, followed by code to put the value of the `define` (which is the symbol `ok`) into the target register, followed finally by the linkage code. `Env` is preserved around the computation of the value, because it is needed in order to install the definition. Because the linkage is `next`, there is no linkage code in this case. The skeleton of the compiled code is thus

``` {.scheme}
⟨save env if modified by code to compute value⟩
  ⟨compilation of definition value, 
   target val, linkage next⟩
  ⟨restore env if saved above⟩
  (perform (op define-variable!)
           (const factorial)
           (reg val)
           (reg env))
  (assign val (const ok))
```

The expression that is to be compiled to produce the value for the variable `factorial` is a `lambda` expression whose value is the procedure that computes factorials. `Compile` handles this by calling `compile-lambda`, which compiles the procedure body, labels it as a new entry point, and generates the instruction that will combine the procedure body at the new entry point with the run-time environment and assign the result to `val`. The sequence then skips around the compiled procedure code, which is inserted at this point. The procedure code itself begins by extending the procedure’s definition environment by a frame that binds the formal parameter `n` to the procedure argument. Then comes the actual procedure body. Since this code for the value of the variable doesn’t modify the `env` register, the optional `save` and `restore` shown above aren’t generated. (The procedure code at `entry2` isn’t executed at this point, so its use of `env` is irrelevant.) Therefore, the skeleton for the compiled code becomes

``` {.scheme}
  (assign val (op make-compiled-procedure)
              (label entry2)
              (reg env))
  (goto (label after-lambda1))
entry2
  (assign env (op compiled-procedure-env)
              (reg proc))
  (assign env (op extend-environment)
              (const (n))
              (reg argl)
              (reg env))
  ⟨compilation of procedure body⟩
after-lambda1
  (perform (op define-variable!)
           (const factorial)
           (reg val) (reg env))
  (assign val (const ok))
```

A procedure body is always compiled (by `compile-lambda-body`) as a sequence with target `val` and linkage `return`. The sequence in this case consists of a single `if` expression:

``` {.scheme}
(if (= n 1)
    1
    (* (factorial (- n 1)) n))
```

`Compile-if` generates code that first computes the predicate (targeted to `val`), then checks the result and branches around the true branch if the predicate is false. `Env` and `continue` are preserved around the predicate code, since they may be needed for the rest of the `if` expression. Since the `if` expression is the final expression (and only expression) in the sequence making up the procedure body, its target is `val` and its linkage is `return`, so the true and false branches are both compiled with target `val` and linkage `return`. (That is, the value of the conditional, which is the value computed by either of its branches, is the value of the procedure.)

``` {.scheme}
⟨save continue, env if modified by 
 predicate and needed by branches⟩
  ⟨compilation of predicate, 
   target val, linkage next⟩
  ⟨restore continue, env if saved above⟩
  (test (op false?) (reg val))
  (branch (label false-branch4))
true-branch5
  ⟨compilation of true branch, 
   target val, linkage return⟩
false-branch4
  ⟨compilation of false branch, 
   target val, linkage return⟩
after-if3
```

The predicate `(= n 1)` is a procedure call. This looks up the operator (the symbol `=`) and places this value in `proc`. It then assembles the arguments `1` and the value of `n` into `argl`. Then it tests whether `proc` contains a primitive or a compound procedure, and dispatches to a primitive branch or a compound branch accordingly. Both branches resume at the `after-call` label. The requirements to preserve registers around the evaluation of the operator and operands don’t result in any saving of registers, because in this case those evaluations don’t modify the registers in question.

``` {.scheme}
  (assign proc (op lookup-variable-value)
               (const =) 
               (reg env))
  (assign val (const 1))
  (assign argl (op list) (reg val))
  (assign val (op lookup-variable-value)
              (const n)
              (reg env))
  (assign argl (op cons) (reg val) (reg argl))
  (test (op primitive-procedure?) (reg proc))
  (branch (label primitive-branch17))
compiled-branch16
  (assign continue (label after-call15))
  (assign val (op compiled-procedure-entry)
              (reg proc))
  (goto (reg val))
primitive-branch17
  (assign val (op apply-primitive-procedure)
              (reg proc)
              (reg argl))
after-call15
```

The true branch, which is the constant 1, compiles (with target `val` and linkage `return`) to

``` {.scheme}
(assign val (const 1))
(goto (reg continue))
```

The code for the false branch is another procedure call, where the procedure is the value of the symbol `*`, and the arguments are `n` and the result of another procedure call (a call to `factorial`). Each of these calls sets up `proc` and `argl` and its own primitive and compound branches. [Figure 5.17](#Figure-5_002e17) shows the complete compilation of the definition of the `factorial` procedure. Notice that the possible `save` and `restore` of `continue` and `env` around the predicate, shown above, are in fact generated, because these registers are modified by the procedure call in the predicate and needed for the procedure call and the `return` linkage in the branches.

> **Figure 5.17:** $\downarrow$ Compilation of the definition of the `factorial` procedure.
> > ``` {.scheme}
> ;; construct the procedure and skip over code
> ;; for the procedure body
> (assign val
> (op make-compiled-procedure) 
> (label entry2) 
> (reg env))
> (goto (label after-lambda1))
> entry2     ; calls to factorial will enter here
> (assign env 
> (op compiled-procedure-env)
> (reg proc))
> (assign env
> (op extend-environment) 
> (const (n)) 
> (reg argl) 
> (reg env))
> ;; begin actual procedure body
> (save continue)
> (save env)
> ;; compute (= n 1)
> (assign proc 
> (op lookup-variable-value) 
> (const =) 
> (reg env))
> (assign val (const 1))
> (assign argl (op list) (reg val))
> (assign val 
> (op lookup-variable-value) 
> (const n) 
> (reg env))
> (assign argl (op cons) (reg val) (reg argl))
> (test (op primitive-procedure?) (reg proc))
> (branch (label primitive-branch17))
> compiled-branch16
> (assign continue (label after-call15))
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch17
> (assign val 
> (op apply-primitive-procedure) 
> (reg proc) 
> (reg argl))
> after-call15   ; val now contains result of (= n 1)
> (restore env)
> (restore continue)
> (test (op false?) (reg val))
> (branch (label false-branch4))
> true-branch5  ; return 1
> (assign val (const 1))
> (goto (reg continue))
> > false-branch4
> ;; compute and return (* (factorial (- n 1)) n)
> (assign proc 
> (op lookup-variable-value) 
> (const *) 
> (reg env))
> (save continue)
> (save proc)   ; save * procedure
> (assign val 
> (op lookup-variable-value) 
> (const n) 
> (reg env))
> (assign argl (op list) (reg val))
> (save argl)   ; save partial argument list for *
> ;; compute (factorial (- n 1)), 
> ;; which is the other argument for *
> (assign proc
> (op lookup-variable-value) 
> (const factorial) 
> (reg env))
> (save proc)  ; save factorial procedure
> ;; compute (- n 1), which is the argument for factorial
> (assign proc 
> (op lookup-variable-value)
> (const -) 
> (reg env))
> (assign val (const 1))
> (assign argl (op list) (reg val))
> (assign val 
> (op lookup-variable-value) 
> (const n) 
> (reg env))
> (assign argl (op cons) (reg val) (reg argl))
> (test (op primitive-procedure?) (reg proc))
> (branch (label primitive-branch8))
> compiled-branch7
> (assign continue (label after-call6))
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch8
> (assign val 
> (op apply-primitive-procedure) 
> (reg proc) 
> (reg argl))
> > after-call6   ; val now contains result of (- n 1)
> (assign argl (op list) (reg val))
> (restore proc) ; restore factorial
> ;; apply factorial
> (test (op primitive-procedure?) (reg proc))
> (branch (label primitive-branch11))
> compiled-branch10
> (assign continue (label after-call9))
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch11
> (assign val 
> (op apply-primitive-procedure) 
> (reg proc) 
> (reg argl))
> after-call9      ; val now contains result 
> ; of (factorial (- n 1))
> (restore argl) ; restore partial argument list for *
> (assign argl (op cons) (reg val) (reg argl))
> (restore proc) ; restore *
> (restore continue)
> ;; apply * and return its value
> (test (op primitive-procedure?) (reg proc))
> (branch (label primitive-branch14))
> compiled-branch13
> ;; note that a compound procedure here
> ;; is called tail-recursively
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch14
> (assign val 
> (op apply-primitive-procedure) 
> (reg proc) 
> (reg argl))
> (goto (reg continue))
> after-call12
> after-if3
> after-lambda1
> ;; assign the procedure to the variable factorial
> (perform (op define-variable!) 
> (const factorial) 
> (reg val) 
> (reg env))
> (assign val (const ok))
> ```

**Exercise 5.33:** Consider the following definition of a factorial procedure, which is slightly different from the one given above:


``` {.scheme}
(define (factorial-alt n)
  (if (= n 1)
      1
      (* n (factorial-alt (- n 1)))))
```


Compile this procedure and compare the resulting code with that produced for `factorial`. Explain any differences you find. Does either program execute more efficiently than the other?

**Exercise 5.34:** Compile the iterative factorial procedure


``` {.scheme}
(define (factorial n)
  (define (iter product counter)
    (if (> counter n)
        product
        (iter (* counter product)
              (+ counter 1))))
  (iter 1 1))
```


Annotate the resulting code, showing the essential difference between the code for iterative and recursive versions of `factorial` that makes one process build up stack space and the other run in constant stack space.

**Exercise 5.35:** What expression was compiled to produce the code shown in [Figure 5.18](#Figure-5_002e18)?

> **Figure 5.18:** $\downarrow$ An example of compiler output. See [Exercise 5.35](#Exercise-5_002e35).
> > ``` {.scheme}
> (assign val (op make-compiled-procedure) 
> (label entry16) 
> (reg env))
> (goto (label after-lambda15))
> entry16
> (assign env (op compiled-procedure-env)
> (reg proc))
> (assign env (op extend-environment) 
> (const (x)) 
> (reg argl) 
> (reg env))
> (assign proc (op lookup-variable-value) 
> (const +) 
> (reg env))
> (save continue) (save proc) (save env)
> (assign proc (op lookup-variable-value) 
> (const g) 
> (reg env))
> (save proc)
> (assign proc (op lookup-variable-value) 
> (const +) 
> (reg env))
> (assign val (const 2))
> (assign argl (op list) (reg val))
> (assign val (op lookup-variable-value)
> (const x) 
> (reg env))
> (assign argl (op cons)
> (reg val)
> (reg argl))
> (test (op primitive-procedure?)
> (reg proc))
> (branch (label primitive-branch19))
> compiled-branch18
> (assign continue (label after-call17))
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch19
> (assign val
> (op apply-primitive-procedure)
> (reg proc) 
> (reg argl))
> after-call17
> (assign argl (op list) (reg val))
> (restore proc)
> (test (op primitive-procedure?)
> (reg proc))
> (branch (label primitive-branch22))
> compiled-branch21
> (assign continue (label after-call20))
> (assign val
> (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch22
> (assign val 
> (op apply-primitive-procedure) 
> (reg proc) 
> (reg argl))
> after-call20
> (assign argl (op list) (reg val))
> (restore env)
> (assign val
> (op lookup-variable-value) 
> (const x) 
> (reg env))
> (assign argl
> (op cons)
> (reg val)
> (reg argl))
> (restore proc)
> (restore continue)
> (test (op primitive-procedure?)
> (reg proc))
> (branch (label primitive-branch25))
> compiled-branch24
> (assign val (op compiled-procedure-entry)
> (reg proc))
> (goto (reg val))
> primitive-branch25
> (assign val 
> (op apply-primitive-procedure)
> (reg proc) 
> (reg argl))
> (goto (reg continue))
> after-call23
> after-lambda15
> (perform (op define-variable!) 
> (const f) 
> (reg val) 
> (reg env))
> (assign val (const ok))
> ```

**Exercise 5.36:** What order of evaluation does our compiler produce for operands of a combination? Is it left-to-right, right-to-left, or some other order? Where in the compiler is this order determined? Modify the compiler so that it produces some other order of evaluation. (See the discussion of order of evaluation for the explicit-control evaluator in [5.4.1](5_002e4.xhtml#g_t5_002e4_002e1).) How does changing the order of operand evaluation affect the efficiency of the code that constructs the argument list?

**Exercise 5.37:** One way to understand the compiler’s `preserving` mechanism for optimizing stack usage is to see what extra operations would be generated if we did not use this idea. Modify `preserving` so that it always generates the `save` and `restore` operations. Compile some simple expressions and identify the unnecessary stack operations that are generated. Compare the code to that generated with the `preserving` mechanism intact.

**Exercise 5.38:** Our compiler is clever about avoiding unnecessary stack operations, but it is not clever at all when it comes to compiling calls to the primitive procedures of the language in terms of the primitive operations supplied by the machine. For example, consider how much code is compiled to compute `(+ a 1)`: The code sets up an argument list in `argl`, puts the primitive addition procedure (which it finds by looking up the symbol `+` in the environment) into `proc`, and tests whether the procedure is primitive or compound. The compiler always generates code to perform the test, as well as code for primitive and compound branches (only one of which will be executed). We have not shown the part of the controller that implements primitives, but we presume that these instructions make use of primitive arithmetic operations in the machine’s data paths. Consider how much less code would be generated if the compiler could *open-code* primitives—that is, if it could generate code to directly use these primitive machine operations. The expression `(+ a 1)` might be compiled into something as simple as^[We have used the same symbol `+` here to denote both the source-language procedure and the machine operation. In general there will not be a one-to-one correspondence between primitives of the source language and primitives of the machine.]


``` {.scheme}
(assign val (op lookup-variable-value) 
            (const a) 
            (reg env))
(assign val (op +)
            (reg val)
            (const 1))
```


In this exercise we will extend our compiler to support open coding of selected primitives. Special-purpose code will be generated for calls to these primitive procedures instead of the general procedure-application code. In order to support this, we will augment our machine with special argument registers `arg1` and `arg2`. The primitive arithmetic operations of the machine will take their inputs from `arg1` and `arg2`. The results may be put into `val`, `arg1`, or `arg2`.

The compiler must be able to recognize the application of an open-coded primitive in the source program. We will augment the dispatch in the `compile` procedure to recognize the names of these primitives in addition to the reserved words (the special forms) it currently recognizes.^[Making the primitives into reserved words is in general a bad idea, since a user cannot then rebind these names to different procedures. Moreover, if we add reserved words to a compiler that is in use, existing programs that define procedures with these names will stop working. See [Exercise 5.44](#Exercise-5_002e44) for ideas on how to avoid this problem.] For each special form our compiler has a code generator. In this exercise we will construct a family of code generators for the open-coded primitives.

1.  The open-coded primitives, unlike the special forms, all need their operands evaluated. Write a code generator `spread-arguments` for use by all the open-coding code generators. `Spread-arguments` should take an operand list and compile the given operands targeted to successive argument registers. Note that an operand may contain a call to an open-coded primitive, so argument registers will have to be preserved during operand evaluation.
2.  For each of the primitive procedures `=`, `*`, `-`, and `+`, write a code generator that takes a combination with that operator, together with a target and a linkage descriptor, and produces code to spread the arguments into the registers and then perform the operation targeted to the given target with the given linkage. You need only handle expressions with two operands. Make `compile` dispatch to these code generators.
3.  Try your new compiler on the `factorial` example. Compare the resulting code with the result produced without open coding.
4.  Extend your code generators for `+` and `*` so that they can handle expressions with arbitrary numbers of operands. An expression with more than two operands will have to be compiled into a sequence of operations, each with only two inputs.


## 5.5.6 Lexical Addressing

One of the most common optimizations performed by compilers is the optimization of variable lookup. Our compiler, as we have implemented it so far, generates code that uses the `lookup-variable-value` operation of the evaluator machine. This searches for a variable by comparing it with each variable that is currently bound, working frame by frame outward through the run-time environment. This search can be expensive if the frames are deeply nested or if there are many variables. For example, consider the problem of looking up the value of `x` while evaluating the expression `(* x y z)` in an application of the procedure that is returned by

``` {.scheme}
(let ((x 3) (y 4))
  (lambda (a b c d e)
    (let ((y (* a b x))
          (z (+ c d x)))
      (* x y z))))
```

Since a `let` expression is just syntactic sugar for a `lambda` combination, this expression is equivalent to

``` {.scheme}
((lambda (x y)
   (lambda (a b c d e)
     ((lambda (y z) (* x y z))
      (* a b x)
      (+ c d x))))
 3
 4)
```

Each time `lookup-variable-value` searches for `x`, it must determine that the symbol `x` is not `eq?` to `y` or `z` (in the first frame), nor to `a`, `b`, `c`, `d`, or `e` (in the second frame). We will assume, for the moment, that our programs do not use `define`—that variables are bound only with `lambda`. Because our language is lexically scoped, the run-time environment for any expression will have a structure that parallels the lexical structure of the program in which the expression appears.^[This is not true if we allow internal definitions, unless we scan them out. See [Exercise 5.43](#Exercise-5_002e43).] Thus, the compiler can know, when it analyzes the above expression, that each time the procedure is applied the variable `x` in `(* x y z)` will be found two frames out from the current frame and will be the first variable in that frame.

We can exploit this fact by inventing a new kind of variable-lookup operation, `lexical-address-lookup`, that takes as arguments an environment and a *lexical address* that consists of two numbers: a *frame number*, which specifies how many frames to pass over, and a *displacement number*, which specifies how many variables to pass over in that frame. `Lexical-address-lookup` will produce the value of the variable stored at that lexical address relative to the current environment. If we add the `lexical-address-lookup` operation to our machine, we can make the compiler generate code that references variables using this operation, rather than `lookup-variable-value`. Similarly, our compiled code can use a new `lexical-address-set!` operation instead of `set-variable-value!`.

In order to generate such code, the compiler must be able to determine the lexical address of a variable it is about to compile a reference to. The lexical address of a variable in a program depends on where one is in the code. For example, in the following program, the address of `x` in expression `⟨`e1`⟩` is (2, 0)—two frames back and the first variable in the frame. At that point `y` is at address (0, 0) and `c` is at address (1, 2). In expression `⟨`e2`⟩`, `x` is at (1, 0), `y` is at (1, 1), and `c` is at (0, 2).

``` {.scheme}
((lambda (x y)
   (lambda (a b c d e)
     ((lambda (y z) ⟨e1⟩)
      ⟨e2⟩
      (+ c d x))))
 3
 4)
```

One way for the compiler to produce code that uses lexical addressing is to maintain a data structure called a *compile-time environment*. This keeps track of which variables will be at which positions in which frames in the run-time environment when a particular variable-access operation is executed. The compile-time environment is a list of frames, each containing a list of variables. (There will of course be no values bound to the variables, since values are not computed at compile time.) The compile-time environment becomes an additional argument to `compile` and is passed along to each code generator. The top-level call to `compile` uses an empty compile-time environment. When a `lambda` body is compiled, `compile-lambda-body` extends the compile-time environment by a frame containing the procedure’s parameters, so that the sequence making up the body is compiled with that extended environment. At each point in the compilation, `compile-variable` and `compile-assignment` use the compile-time environment in order to generate the appropriate lexical addresses.

[Exercise 5.39](#Exercise-5_002e39) through [Exercise 5.43](#Exercise-5_002e43) describe how to complete this sketch of the lexical-addressing strategy in order to incorporate lexical lookup into the compiler. [Exercise 5.44](#Exercise-5_002e44) describes another use for the compile-time environment.

**Exercise 5.39:** Write a procedure `lexical-address-lookup` that implements the new lookup operation. It should take two arguments—a lexical address and a run-time environment—and return the value of the variable stored at the specified lexical address. `Lexical-address-lookup` should signal an error if the value of the variable is the symbol `*unassigned*`.^[This is the modification to variable lookup required if we implement the scanning method to eliminate internal definitions ([Exercise 5.43](#Exercise-5_002e43)). We will need to eliminate these definitions in order for lexical addressing to work.] Also write a procedure `lexical-address-set!` that implements the operation that changes the value of the variable at a specified lexical address.

**Exercise 5.40:** Modify the compiler to maintain the compile-time environment as described above. That is, add a compile-time-environment argument to `compile` and the various code generators, and extend it in `compile-lambda-body`.

**Exercise 5.41:** Write a procedure `find-variable` that takes as arguments a variable and a compile-time environment and returns the lexical address of the variable with respect to that environment. For example, in the program fragment that is shown above, the compile-time environment during the compilation of expression `⟨`e1`⟩` is `((y z) (a b c d e) (x y))`. `Find-variable` should produce


``` {.scheme}
(find-variable 
 'c '((y z) (a b c d e) (x y)))
(1 2)

(find-variable 
 'x '((y z) (a b c d e) (x y)))
(2 0)

(find-variable 
 'w '((y z) (a b c d e) (x y)))
not-found
```


**Exercise 5.42:** Using `find-variable` from [Exercise 5.41](#Exercise-5_002e41), rewrite `compile-variable` and `compile-assignment` to output lexical-address instructions. In cases where `find-variable` returns `not-found` (that is, where the variable is not in the compile-time environment), you should have the code generators use the evaluator operations, as before, to search for the binding. (The only place a variable that is not found at compile time can be is in the global environment, which is part of the run-time environment but is not part of the compile-time environment.^[Lexical addresses cannot be used to access variables in the global environment, because these names can be defined and redefined interactively at any time. With internal definitions scanned out, as in [Exercise 5.43](#Exercise-5_002e43), the only definitions the compiler sees are those at top level, which act on the global environment. Compilation of a definition does not cause the defined name to be entered in the compile-time environment.] Thus, if you wish, you may have the evaluator operations look directly in the global environment, which can be obtained with the operation `(op get-global-environment)`, instead of having them search the whole run-time environment found in `env`.) Test the modified compiler on a few simple cases, such as the nested `lambda` combination at the beginning of this section.

**Exercise 5.43:** We argued in [4.1.6](4_002e1.xhtml#g_t4_002e1_002e6) that internal definitions for block structure should not be considered “real” `define`s. Rather, a procedure body should be interpreted as if the internal variables being defined were installed as ordinary `lambda` variables initialized to their correct values using `set!`. [4.1.6](4_002e1.xhtml#g_t4_002e1_002e6) and [Exercise 4.16](4_002e1.xhtml#Exercise-4_002e16) showed how to modify the metacircular interpreter to accomplish this by scanning out internal definitions. Modify the compiler to perform the same transformation before it compiles a procedure body.

**Exercise 5.44:** In this section we have focused on the use of the compile-time environment to produce lexical addresses. But there are other uses for compile-time environments. For instance, in [Exercise 5.38](#Exercise-5_002e38) we increased the efficiency of compiled code by open-coding primitive procedures. Our implementation treated the names of open-coded procedures as reserved words. If a program were to rebind such a name, the mechanism described in [Exercise 5.38](#Exercise-5_002e38) would still open-code it as a primitive, ignoring the new binding. For example, consider the procedure


``` {.scheme}
(lambda (+ * a b x y)
  (+ (* a x) (* b y)))
```


which computes a linear combination of `x` and `y`. We might call it with arguments `+matrix`, `*matrix`, and four matrices, but the open-coding compiler would still open-code the `+` and the `*` in `(+ (* a x) (* b y))` as primitive `+` and `*`. Modify the open-coding compiler to consult the compile-time environment in order to compile the correct code for expressions involving the names of primitive procedures. (The code will work correctly as long as the program does not `define` or `set!` these names.)


## 5.5.7 Interfacing Compiled Code to the Evaluator

We have not yet explained how to load compiled code into the evaluator machine or how to run it. We will assume that the explicit-control-evaluator machine has been defined as in [5.4.4](5_002e4.xhtml#g_t5_002e4_002e4), with the additional operations specified in [Footnote 323](#Footnote-323). We will implement a procedure `compile-and-go` that compiles a Scheme expression, loads the resulting object code into the evaluator machine, and causes the machine to run the code in the evaluator global environment, print the result, and enter the evaluator’s driver loop. We will also modify the evaluator so that interpreted expressions can call compiled procedures as well as interpreted ones. We can then put a compiled procedure into the machine and use the evaluator to call it:

``` {.scheme}
(compile-and-go
 '(define (factorial n)
    (if (= n 1)
        1
        (* (factorial (- n 1)) n))))

;;; EC-Eval value:
ok

;;; EC-Eval input:
(factorial 5)

;;; EC-Eval value:
120
```

To allow the evaluator to handle compiled procedures (for example, to evaluate the call to `factorial` above), we need to change the code at `apply-dispatch` ([5.4.1](5_002e4.xhtml#g_t5_002e4_002e1)) so that it recognizes compiled procedures (as distinct from compound or primitive procedures) and transfers control directly to the entry point of the compiled code:^[Of course, compiled procedures as well as interpreted procedures are compound (nonprimitive). For compatibility with the terminology used in the explicit-control evaluator, in this section we will use “compound” to mean interpreted (as opposed to compiled).]

``` {.scheme}
apply-dispatch
  (test (op primitive-procedure?) (reg proc))
  (branch (label primitive-apply))
  (test (op compound-procedure?) (reg proc))
  (branch (label compound-apply))
  (test (op compiled-procedure?) (reg proc))
  (branch (label compiled-apply))
  (goto (label unknown-procedure-type))

compiled-apply
  (restore continue)
  (assign val
          (op compiled-procedure-entry)
          (reg proc))
  (goto (reg val))
```

Note the restore of `continue` at `compiled-apply`. Recall that the evaluator was arranged so that at `apply-dispatch`, the continuation would be at the top of the stack. The compiled code entry point, on the other hand, expects the continuation to be in `continue`, so `continue` must be restored before the compiled code is executed.

To enable us to run some compiled code when we start the evaluator machine, we add a `branch` instruction at the beginning of the evaluator machine, which causes the machine to go to a new entry point if the `flag` register is set.^[Now that the evaluator machine starts with a `branch`, we must always initialize the `flag` register before starting the evaluator machine. To start the machine at its ordinary read-eval-print loop, we could use]

``` {.scheme}
;; branches if flag is set:
(branch (label external-entry)) 
read-eval-print-loop
  (perform (op initialize-stack))
  …
```

`External-entry` assumes that the machine is started with `val` containing the location of an instruction sequence that puts a result into `val` and ends with `(goto (reg continue))`. Starting at this entry point jumps to the location designated by `val`, but first assigns `continue` so that execution will return to `print-result`, which prints the value in `val` and then goes to the beginning of the evaluator’s read-eval-print loop.^[Since a compiled procedure is an object that the system may try to print, we also modify the system print operation `user-print` (from [4.1.4](4_002e1.xhtml#g_t4_002e1_002e4)) so that it will not attempt to print the components of a compiled procedure:]

``` {.scheme}
external-entry
  (perform (op initialize-stack))
  (assign env (op get-global-environment))
  (assign continue (label print-result))
  (goto (reg val))
```

Now we can use the following procedure to compile a procedure definition, execute the compiled code, and run the read-eval-print loop so we can try the procedure. Because we want the compiled code to return to the location in `continue` with its result in `val`, we compile the expression with a target of `val` and a linkage of `return`. In order to transform the object code produced by the compiler into executable instructions for the evaluator register machine, we use the procedure `assemble` from the register-machine simulator ([5.2.2](5_002e2.xhtml#g_t5_002e2_002e2)). We then initialize the `val` register to point to the list of instructions, set the `flag` so that the evaluator will go to `external-entry`, and start the evaluator.

``` {.scheme}
(define (compile-and-go expression)
  (let ((instructions
         (assemble 
          (statements
           (compile 
            expression 'val 'return))
          eceval)))
    (set! the-global-environment
          (setup-environment))
    (set-register-contents! 
     eceval 'val instructions)
    (set-register-contents! 
     eceval 'flag true)
    (start eceval)))
```

If we have set up stack monitoring, as at the end of [5.4.4](5_002e4.xhtml#g_t5_002e4_002e4), we can examine the stack usage of compiled code:

``` {.scheme}
(compile-and-go
 '(define (factorial n)
    (if (= n 1)
        1
        (* (factorial (- n 1)) n))))
(total-pushes = 0, maximum-depth = 0)

;;; EC-Eval value:
ok

;;; EC-Eval input:
(factorial 5)
(total-pushes = 31, maximum-depth = 14)

;;; EC-Eval value:
120
```

Compare this example with the evaluation of `(factorial 5)` using the interpreted version of the same procedure, shown at the end of [5.4.4](5_002e4.xhtml#g_t5_002e4_002e4). The interpreted version required 144 pushes and a maximum stack depth of 28. This illustrates the optimization that results from our compilation strategy.


### Interpretation and compilation

With the programs in this section, we can now experiment with the alternative execution strategies of interpretation and compilation.^[We can do even better by extending the compiler to allow compiled code to call interpreted procedures. See [Exercise 5.47](#Exercise-5_002e47).] An interpreter raises the machine to the level of the user program; a compiler lowers the user program to the level of the machine language. We can regard the Scheme language (or any programming language) as a coherent family of abstractions erected on the machine language. Interpreters are good for interactive program development and debugging because the steps of program execution are organized in terms of these abstractions, and are therefore more intelligible to the programmer. Compiled code can execute faster, because the steps of program execution are organized in terms of the machine language, and the compiler is free to make optimizations that cut across the higher-level abstractions.^[Independent of the strategy of execution, we incur significant overhead if we insist that errors encountered in execution of a user program be detected and signaled, rather than being allowed to kill the system or produce wrong answers. For example, an out-of-bounds array reference can be detected by checking the validity of the reference before performing it. The overhead of checking, however, can be many times the cost of the array reference itself, and a programmer should weigh speed against safety in determining whether such a check is desirable. A good compiler should be able to produce code with such checks, should avoid redundant checks, and should allow programmers to control the extent and type of error checking in the compiled code.]

The alternatives of interpretation and compilation also lead to different strategies for porting languages to new computers. Suppose that we wish to implement Lisp for a new machine. One strategy is to begin with the explicit-control evaluator of [5.4](5_002e4.xhtml#g_t5_002e4) and translate its instructions to instructions for the new machine. A different strategy is to begin with the compiler and change the code generators so that they generate code for the new machine. The second strategy allows us to run any Lisp program on the new machine by first compiling it with the compiler running on our original Lisp system, and linking it with a compiled version of the run-time library.^[Of course, with either the interpretation or the compilation strategy we must also implement for the new machine storage allocation, input and output, and all the various operations that we took as “primitive” in our discussion of the evaluator and compiler. One strategy for minimizing work here is to write as many of these operations as possible in Lisp and then compile them for the new machine. Ultimately, everything reduces to a small kernel (such as garbage collection and the mechanism for applying actual machine primitives) that is hand-coded for the new machine.] Better yet, we can compile the compiler itself, and run this on the new machine to compile other Lisp programs.^[This strategy leads to amusing tests of correctness of the compiler, such as checking whether the compilation of a program on the new machine, using the compiled compiler, is identical with the compilation of the program on the original Lisp system. Tracking down the source of differences is fun but often frustrating, because the results are extremely sensitive to minuscule details.] Or we can compile one of the interpreters of [4.1](4_002e1.xhtml#g_t4_002e1) to produce an interpreter that runs on the new machine.

**Exercise 5.45:** By comparing the stack operations used by compiled code to the stack operations used by the evaluator for the same computation, we can determine the extent to which the compiler optimizes use of the stack, both in speed (reducing the total number of stack operations) and in space (reducing the maximum stack depth). Comparing this optimized stack use to the performance of a special-purpose machine for the same computation gives some indication of the quality of the compiler.

1.  [Exercise 5.27](5_002e4.xhtml#Exercise-5_002e27) asked you to determine, as a function of $n$, the number of pushes and the maximum stack depth needed by the evaluator to compute $n!$ using the recursive factorial procedure given above. [Exercise 5.14](5_002e2.xhtml#Exercise-5_002e14) asked you to do the same measurements for the special-purpose factorial machine shown in [Figure 5.11](5_002e1.xhtml#Figure-5_002e11). Now perform the same analysis using the compiled `factorial` procedure.

    Take the ratio of the number of pushes in the compiled version to the number of pushes in the interpreted version, and do the same for the maximum stack depth. Since the number of operations and the stack depth used to compute $n!$ are linear in $n$, these ratios should approach constants as $n$ becomes large. What are these constants? Similarly, find the ratios of the stack usage in the special-purpose machine to the usage in the interpreted version.

    Compare the ratios for special-purpose versus interpreted code to the ratios for compiled versus interpreted code. You should find that the special-purpose machine does much better than the compiled code, since the hand-tailored controller code should be much better than what is produced by our rudimentary general-purpose compiler.

2.  Can you suggest improvements to the compiler that would help it generate code that would come closer in performance to the hand-tailored version?

**Exercise 5.46:** Carry out an analysis like the one in [Exercise 5.45](#Exercise-5_002e45) to determine the effectiveness of compiling the tree-recursive Fibonacci procedure


``` {.scheme}
(define (fib n)
  (if (< n 2)
      n
      (+ (fib (- n 1)) (fib (- n 2)))))
```


compared to the effectiveness of using the special-purpose Fibonacci machine of [Figure 5.12](5_002e1.xhtml#Figure-5_002e12). (For measurement of the interpreted performance, see [Exercise 5.29](5_002e4.xhtml#Exercise-5_002e29).) For Fibonacci, the time resource used is not linear in $n;$ hence the ratios of stack operations will not approach a limiting value that is independent of $n$.

**Exercise 5.47:** This section described how to modify the explicit-control evaluator so that interpreted code can call compiled procedures. Show how to modify the compiler so that compiled procedures can call not only primitive procedures and compiled procedures, but interpreted procedures as well. This requires modifying `compile-procedure-call` to handle the case of compound (interpreted) procedures. Be sure to handle all the same `target` and `linkage` combinations as in `compile-proc-appl`. To do the actual procedure application, the code needs to jump to the evaluator’s `compound-apply` entry point. This label cannot be directly referenced in object code (since the assembler requires that all labels referenced by the code it is assembling be defined there), so we will add a register called `compapp` to the evaluator machine to hold this entry point, and add an instruction to initialize it:


``` {.scheme}
  (assign compapp (label compound-apply))
  ;; branches if flag is set:
  (branch (label external-entry))
read-eval-print-loop …
```


To test your code, start by defining a procedure `f` that calls a procedure `g`. Use `compile-and-go` to compile the definition of `f` and start the evaluator. Now, typing at the evaluator, define `g` and try to call `f`.

**Exercise 5.48:** The `compile-and-go` interface implemented in this section is awkward, since the compiler can be called only once (when the evaluator machine is started). Augment the compiler-interpreter interface by providing a `compile-and-run` primitive that can be called from within the explicit-control evaluator as follows:


``` {.scheme}
;;; EC-Eval input:
(compile-and-run
 '(define (factorial n)
    (if (= n 1)
        1
        (* (factorial (- n 1)) n))))

;;; EC-Eval value:
ok

;;; EC-Eval input:
(factorial 5)

;;; EC-Eval value:
120
```


**Exercise 5.49:** As an alternative to using the explicit-control evaluator’s read-eval-print loop, design a register machine that performs a read-compile-execute-print loop. That is, the machine should run a loop that reads an expression, compiles it, assembles and executes the resulting code, and prints the result. This is easy to run in our simulated setup, since we can arrange to call the procedures `compile` and `assemble` as “register-machine operations.”

**Exercise 5.50:** Use the compiler to compile the metacircular evaluator of [4.1](4_002e1.xhtml#g_t4_002e1) and run this program using the register-machine simulator. (To compile more than one definition at a time, you can package the definitions in a `begin`.) The resulting interpreter will run very slowly because of the multiple levels of interpretation, but getting all the details to work is an instructive exercise.

**Exercise 5.51:** Develop a rudimentary implementation of Scheme in C (or some other low-level language of your choice) by translating the explicit-control evaluator of [5.4](5_002e4.xhtml#g_t5_002e4) into C. In order to run this code you will need to also provide appropriate storage-allocation routines and other run-time support.

**Exercise 5.52:** As a counterpoint to [Exercise 5.51](#Exercise-5_002e51), modify the compiler so that it compiles Scheme procedures into sequences of C instructions. Compile the metacircular evaluator of [4.1](4_002e1.xhtml#g_t4_002e1) to produce a Scheme interpreter written in C.
