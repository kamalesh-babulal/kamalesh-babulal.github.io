# Linux Kernel Livepatch -mprofile-kernel

Live Patching of Linux Kernel on PowerPC (https://kamalesh-babulal.github.io/LKP-Quick-Intro.html), takes advantage of `-mprofile-kernel` gcc switch. Consider very simple code below, which just calls leaf function foo from main (leaf functions are the once, those does not calls another functions, like the leaf node of any tree).

```
# cat test.c
void foo() { }

int main(void)
{
 foo();
 return (0);
}
```
`# gcc -S test.c -pg`
`-S` switch generates assembly code, while  `-pg`  added to generate profiling information. gcc manual explains in detail the -pg/-S switch, along with exclusive/inclusive options those can be combined with them.

```
 1 main:
 2 0:    addis 2,12,.TOC.-0b@ha
 3       addi 2,2,.TOC.-0b@l
 4       .localentry main,.-main
 5       mflr 0
 6       std 0,16(1)
 7       std 31,-8(1)
 8       stdu 1,-112(1)
 9       mr 31,1
10       bl _mcount
11       nop
12       bl foo
13       li 9,0
14       mr 3,9
15       addi 1,31,112
16       ld 0,16(1)
17       mtlr 0
18       ld 31,-8(1)
19       blr
```
Assembly instructions of Lines 5 to 9, does the setting up of stack before calling `_mcount`. `_mcount` routine is part of profiling code, where the address of caller()/callee() are examined from stack to account for that branch. i.e. to account for how many number of times the function gets called.  One example of knowing the number of times a function gets called  is finding the hot path.  Important step is to setting up of stack frame or stack before calling the profiling function `_mcount`. Profiling function name might `mcount` or  `_mcount` depending upon the platform and compiler used.

`# gcc -S test.c -pg -mprofile-kernel`
```
 1 main:
 2 0:     addis 2,12,.TOC.-0b@ha
 3        addi 2,2,.TOC.-0b@l
 4        .localentry main,.-main
 5        mflr 0
 6        std 0,16(1)
 7        bl _mcount
 8        mflr 0
 9        std 0,16(1)
10        std 31,-8(1)
11        stdu 1,-112(1)
12        mr 31,1
13        bl foo
14        li 9,0
15        mr 3,9
16        addi 1,31,112
17        ld 0,16(1)
18        mtlr 0
19        ld 31,-8(1)
20        blr
```
when `-mprofile-kernel` switch is used along with `-pg` to compile the program, the stack frame is not setup before calling the profiling function, like the way it’s done while `-pg` switch is used.  Line 5 to 7 in the above code, just the link register value (where the return address is available) is pushed to active stack space reserved for return address and profiling function `_mcount` is called.

Now that the `-mprofile-kernel switch` provides, `_mcount` with access to registers and its values, which are holds in stack. Live Patching leverages the access to these address (caller/callee address) by modifying them to call different function and to return from `_mcount` to call function which called the function which in turn called _mcount, by the above example `_mcount` from main could call patched `foo()` and return to `main()`. This is possible as _mcount is called at the function prologue.
