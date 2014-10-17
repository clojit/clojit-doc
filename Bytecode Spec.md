Clojure Bytecode Spec
======================

    B   C   A   OP
    D       A   OP

## Tables

- Global Symbol Table
- Constant Int Table
- Constant Float Table
- Constant String Table
- Constant Keyword Table

## Constant Table Value Operation

    OP      A       D

    CSTR    dst     str
    CKEY    dst     keyword
    CINT    dst     int
    CFLOAT  dst     float
    CTYPE   dst     type

Reads D from const. table (if required) and writes it into destination slot A.

## Constant Value Operation

    OP      A       D

    CBOOL   dst     bool (1 true, 0 false)
    CNIL    dst     -

Set A to constant

    OP      A       D

    CSHORT	dst	    lits

Set A to 16 bit signed integer D



## Global Table Ops

    OP       A       D

    NSSETS   var     const-str    ns[const-str] = var
    NSGETS   dst     const-str    dst = ns[const-str]


NSSETS creates top level definition called by constant table string D with value from A

NSGETS reades top level definition called D and copys it to A


## Variable Slots

All variables are stored in a function local variable slot. TODO: Figure out how
this is supposed to work for function calls.

ISSUE: In the current design, the variables are accessed using a 8-bit index. This
limits the number of local variables to 256. This is not a problem for Lua, as it
allows no more than 200 local variables, but might be a problem for a general
programming language such as Clojure.

## Binary Ops

    OP      A   B   C
    ADDVV	dst	var	var	    A = B + C
    SUBVV	dst	var	var	    A = B - C
    MULVV	dst	var	var	    A = B * C
    DIVVV	dst	var	var	    A = B / C
    MODVV	dst	var	var	    A = B % C
    POWVV dst	var	var	    A = B ^ C

    ISLT	dst var	var	    A = B < C
    ISGE	dst var	var     A = B ≥ C
    ISLE	dst var var	    A = B ≤ C
    ISGT	dst var var	    A = B > C
    ISEQ	dst var var	    A = B == C
    ISNEQ	dst var var	    A = B ≠ C

Executes the operation on the numbers B and C.
For the math ops: If one of the operands is a float, the result is of type float. If both operands are longs, the result is of type long.

For the comparision ops: If the operands are not of the same type, the other value is converted to float for comparison. Equals is only true if both operands have the same type.

## Unary Ops

    OP      A       D
    MOV	    dst	    var	    Copy D to A
    NOT	    dst	    var	    Set A to boolean not of D
    NEG	    dst	    var	    Set A to -D (negate)

## Jumps

    OP      A       D
    JUMP    -       addr
    JUMPF   var     addr
    JUMPT   var     addr

Jumps to the relative target address `addr` if the value in slot `var` is truthy or falsy. The `addr` is in amount of bytecode instructions.

## Function Calls

    OP      A       D
    CALL    base    lit
    RET     var     -

'lit' is the number of arguments, including the callee function (i.e. number
of arguments plus one)

base is a offset on the variable belt, so that the slot nr 'base+1' is a reference
to the function, 'base+2' is the first argument, 'base+lit' is the last
argument and so on. The return value of the call will be storred in 'base':


        clojure code: (let [ret (f arg0 arg1 arg2)])
        call instruction: CALL base 4

               +------------+             (callee view)
               |   other    |
               +------------+             +------------+
    base       |    ret     |           0 |    ret     |
               +------------+   ---->     +------------+
    base + 1   |    func    |           1 |    func    |
               +------------+             +------------+
    base + 2   |    arg0    |           2 |    arg0    |
               +------------+             +------------+
    base + 3   |    arg1    |           3 |    arg1    |
               +------------+             +------------+
    base + lit |    arg2    |           4 |    arg2    |
               +------------+             +------------+
                                        5 |   locals   |
                (caller view)             +------------+


The CALL instruction will set up the variable belt for the callee so that
all parameters are in the right place. This means that for the callee its
return address is in slot 0, the function object for itself in slot 1 and its
arguments in slot 2 and following slots. In other words: slot index 'base'
of the caller will become slot index 0 for the callee.

Data stored in slots after 'base+lit' will be overwritten by the call.

The RET instruction will copy the value in `var` into the designated slot for
the caller. RET will read out the return address from slot 0, replace it with
the contents of `var` and then jump to the return address.

    OP      A       B       C
    APPLY   var     var     -

Calls the function stored on variable slot A, and applies the elements from
vector B as argument for the function.

## Closures and Free Variables

    OP       A       D
    FNEW     dst     index
    VFNEW    dst     index

FNEW creates new closure for the function referenced by `jump` in the variable
slot `dst`.

    OP          A       D
    GETFREEVAR  dst     idx

GETFREEVAR copies the value of the free variable `idx` into the variable in `dst`.

    OP          A       D
    UCLO        sslot   eslot

Close upvalues for slots between sslot and eslot


## Tail Recursion and Loops

    OP          A       B       C
    LOOP        -       -       -
    BULKMOV     dst     src     len

LOOP is a no-op to indicate the beginning of a loop. At the end of a loop,
a JUMP instruction is used to jump back to the beginning of the loop (i.e. where
the LOOP instruction is). BULKMOV copies `len` variables starting from `src` into
`dst`, `dst+1`, ..., `dst+len`. BULKMOV is used to reset the loop variables.

Tail recursion is implemented using the JUMP instruction, the function jumps
back to the start of the function. The compiler emits the code to set-up the
new arguments of the function.

## Arrays

    OP          A       D

    NEWARRAY    dst     size

    OP          A       B       C

    GETARRAY    dst     array   idx     dst = array[idx]
    SETARRAY    array   idx     val     array[idx] =  val

Creates a new array of size `size` in the variable slot `dst`. The element size
of arrays is 64bit. The array index `idx` is read from a variable slot, so is the `size`.

## Function Def

    OP      A       D
    FUNCF   lit     -
    FUNCV   lit     -

FUNCF & FUNCV define a function with 'lit' fixed arguments. The FUNCV opcode
defines a function which has an additional vararg argument.
(defn [a b & c] ...) == FUNCV 2

## Types

    OP       A       D

    ALLOC    dst     type

ALLOC allocates the empty instance of Type D and puts a reference to into dst.

    OP        A    B            C

    SETFIELD  ref  offset(lit)  var

SETFIELD writes var to the correct slot in instance referenced by A.


    OP        A    D

    GETFIELD  ref  offset(lit)

GETFIELD writes var to the correct slot in instance referenced by A.


## Run-Time Behavior

    OP      A       D
    BREAK   -       -
    EXIT    lit     -
    DROP    var     var

BREAK triggers a breakpoint. EXIT terminates the virtual machine, the variable
in slot `var` indicates the status code. Nonzero status codes shall indicate
abnormal termination.

DROP tells the VM that it can GC the values in slots A - D

## TODO

 * Define type layout and creation (vtables).
 * Representation of stack values: pointers, integers (NaN tagging).
 * Define byte code for defining and creating protocols.
 * Runtime extension of types (reify).

# Type Tagging

Without using a shadow stack to store the type of register values, the only
way to store the type is to reserve some bits for it. With tagging, there is
no option to actually store 64 bit integers in registers, as some bits have
to be used for the type.

Current operating systems on x86-64 enforce a pointer structure where the upper
16 bits and the lower 3 bits of an pointer are forced to be zero, [see Wikipedia
for more details](https://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details)

Since there is no way to represent 64 bit integers, the following properties
would be nice, regarding to unboxing (i.e. using the register value for directly
in an operation without any bit modifications to make it fit):

1. Fast 32bit integers, without the need for unboxing the values first. This is
   possible on x86-64 if only the upper 32 bits are used for tagging, since x86
   allows direct operations on the lower 32 bits.
2. Fast pointer access on x86-64. Since a valid virtual address is only 48 bits
   wide, the upper 16 bits can be used for tagging. If the pointer tag is all-zero,
   then no unboxing would be needed.
3. Fast double-precision floating point access. Since there are 2^51 NaN values, but
   only one is used in practise, it is possible to implement fast float operations
   without unboxing.
4. Use additional tag bits not only to determine that a register value is a pointer,
   but also to determine which type of pointer it is, e.g. to differentiate between
   objects pointers and functions pointers.

Property (1.) cannot be achieved if the lower bits are used for tagging. V8 uses
the lower bits for tagging, but seems to have an optimizing compiler to avoid
unnecessary boxing/unboxing operations.

Having property (2.) means that pointers can be dereferenced without unboxing.
Property (2.) is in conflict with property (3.), since a valid pointer value can
also be a valid float value, therefore float values have to be unboxed.
Property (2.) can also not be directly combined with property (4.), since it is
not possible to store any additional information about the pointer without
modifying it.

Property (2.) seems to help with garbage collection, since the garbage collector
does not have to perform any modifications on the pointer to resolve it.

Properties (1.), (2.) and (4.) can be achieved without conflict using NaN boxing.
This means that only for pointer access the values have to be unboxed before use.
At least on x86-64, floating point and 32-bit integer operations can be performed
directly on the register, without modifying bit operations.
Only pointers have to be unboxed before use, but the 48 bit pointers of x86-64
are small enough to be stored in the 51 bit fraction of a NaN double.


Resources
---------

###LuaJit:
http://wiki.luajit.org/Bytecode-2.0
http://wiki.luajit.org/Optimizations

###Java:
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.2
http://en.wikipedia.org/wiki/Java_bytecode

###Clojure:
http://clojure.org/special_forms
http://clojure.org/datatypes
http://clojuredocs.org/clojure_core/clojure.core/deftype

https://github.com/clojure/clojurescript/blob/master/src/clj/cljs/compiler.clj
https://github.com/clojure/clojurescript/blob/master/src/clj/cljs/core.clj

https://github.com/clojure/tools.reader
https://github.com/clojure/tools.analyzer

https://github.com/halgari/clojure-py

http://clojure-py.blogspot.de/

###Guile

With Guile 2.2 they will have a VM with Register Bytecode, see:
http://wingolog.org/archives/2013/11/26/a-register-vm-for-guile


Problems:
----------

Clojure was designed to run on a host. Both JVM Clojure and ClojureScript (JS) assume some functionality to be present on the host platform.

These features either have to be inplmented in Clojure ontop of the current special froms or we have to provide native inpmnetation (clojure-py).

Ideas:
---------

There is only one looping construct in clojure, yet we often have more infomration from standard macros (dotimes ...)
. Maybe we can somehow use these information and give to the VM. We also have information about the length of loops over collection (map inc [1 2 3]). We have constant access to the size of persistent collection, this information might help the VM.



Desings:
--------
