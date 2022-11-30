---
title: "Assembly of the Mad"
date: 2022-11-29T15:08:26+01:00
author: Martin Kauppinen
---

This past weekend, I participated in [Glacier CTF](https://ctf.glacierctf.com/),
and had a lot of fun, though I did get stuck on a particular problem for far too
long without managing to solve it. But I did have some fun attempting to do so!
I suppose this post is both a cautionary tale to not get too hung up on what you
think the way forward is (especially if you think the way forward is
hand-writing x86 assembly), and a documentation of how to do syscalls manually
in x86 assembly.

{{< toc >}}

## The task
Below is the problem statement as given in the CTF:

> Steves company has a custom service that executes binaries in a sandbox. The
> admins changed his password though - can you still get in and leak theirs as
> revenge?
>
> nc pwn.glacierctf.com 13374

Along with this was a compiled binary called `sandbox`.

When connecting to the server, the first prompt was to input a payload encoded
in [base64](https://en.wikipedia.org/wiki/Base64). After that, there was a
prompt for a username and a password. Inputting the wrong username terminated
the connection, saying that the user was not found. Well, we know from the task
that the service is run by Steve, so `steve` would be a good first attempt at a
username. Here is the result of running such an attempt:
```
$ nc pwn.glacierctf.com 13374
Please enter the base64 encoded payload
Maximum decoded size is 32kb
 
Username: steve
Password: password
password != bf1mNC/KawRK6/cHRtX2JtAr8MNIYLvo6CGabdc6oyg=
```

## The attempt
So now we know that the password must match some hash, taken from somewhere. At
this point, I directed my attention to the provided binary. After decompiling it
with [`ghidra`](https://ghidra-sre.org) and looking through the code, I found
out that this was the login binary that gave the output above, minus the base64
payload part. The program opened a text file containing usernames and password
hashes, and checked the input against them. If an incorrect username was given,
it simply said "User not found", and if an incorrect password was given, it
informed that the password did not match the stored hash. It even gave the exact
hash it had stored for the user. However, the program checked whether you tried
to log in as `admin` and simply exited if you did, without leaking the password
hash.

After painstakingly renaming mangled variable and function names, and mentally
stepping through the code, I finally figured out that the hashing process was
simply doing some swapping and xor:ing of bytes, followed by AES encryption.
However, the xor:ed values and AES encryption keys were simply hardcoded in the
binary, so with some python code I could simply do the encryption and xor-ing
backwards until I got the right password: `H$g7FAKVR8f3&k!@wmMd6Vdk3rHSUrwg`.
```
Please enter the base64 encoded payload
Maximum decoded size is 32kb

Username: steve
Password: H$g7FAKVR8f3&k!@wmMd6Vdk3rHSUrwg
/tmp/tmpfsez12xh
Executing payload...


Payload could not be executed
Is it self-contained?
```

That final line is what started my descent into madness. The base64-encoded
payload that you send to the server before logging in must be
self-contained. Meaning that it cannot dynamically link to anything. And it has
to be less than 32kb in size. That's not a whole lot of room for a `libc`, for
example.

I knew that the login program read the usernames and password hashes from a file
called `userList.txt`. So if I could just open that file and print it, the task
would be done, as I could just reverse the password hash for the admin. However,
before executing the payload, the `sandbox` binary `chroot`s into a temporary
directory. I don't know why, but I figured there might be some other files there
that could be of use, and after trying and failing several times to write a
directory lister in both Go (due to the static linking) and C -- passing the
`-static` flag to `gcc` -- I decided to just do it manually, and hand-write an
assembly program that lists all files in the current directory. Because the Go
and C versions could not fit in the required 32kb with static linking.

## Madness: some assembly required
So, it's time to implement a crude `ls(1)` in x86_64 assembly. But without a
standard library. So we're going to have to use syscalls!

On x86_64, arguments to the `syscall` instruction are given in the `rax`, `rdi`,
`rsi`, `rdx`, `r10`, `r8`, `r9` registers in that order. The return value is
stored in `rax`. Some syscalls also clobber other registers, so be mindful of
that when invoking them[^clobber].
[^clobber]: I noticed that the `open` and `write` syscalls clobbered the `rcx`
  register, which I was initially using in other parts of my code. This was very
  frustrating.

There are many list of the Linux syscalls to use as a reference. I used [this one](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86_64-64_bit).
The important syscall here is `getdents64`, which in C has the following
signature:
```c
ssize_t getdents64(int fd, void *dirp, size_t count);
```
It fills in a list of directory entries at the address pointed to by `dirp` by
reading the file descriptor `fd`. The directory entry struct looks like this:
```c
struct linux_dirent64 {
    uint64_t d_ino;
    int64_t  d_off;
    uint16_t d_reclen;
    uint8_t  d_type;
    char     d_name[];
};
```
The important fields are `d_reclen`, which gives the size of the directory
entry, and `d_name` which is, you guessed it, the name of the directory entry.
So a filename, directory name, link name, etc.

For convenience, we can translate this struct to `nasm` assembly as follows:
```asm
struc dirent64
    .inode:  resb 8
    .offset: resb 8
    .reclen: resb 2
    .type:   resb 1
    .name:   resb 0
endstruc
```
This will make it easier to refer to the offsets correctly.

### Opening a file descriptor
In order to invoke the `getdents64` syscall, I needed a file descriptor for the
current directory. Luckily, this is what the `open` syscall is for. We can begin
the program like so:

```asm
global _start
section .text

struc dirent64
    .inode:  resb 8
    .offset: resb 8
    .reclen: resb 2
    .type:   resb 1
    .name:   resb 0
endstruc

_start:
    call open           ; rax = fd of current directory
    call exit

open:
    mov rax, 2          ; open syscall
    mov rdi, cwd        ; path to open (".")
    mov rsi, 0          ; O_RDONLY
    syscall
    ret

exit:
    mov rax, 60         ; exit syscall
    xor rdi, rdi        ; return value (0)
    syscall

section .rodata
    cwd: db ".", 0
```
This program now opens and gets a file descriptor for the current directory,
then exits with a return code of 0. That doesn't do much, but it's a start.

### Getting dirents
Now we can actually call the `getdents64` syscall and instruct it to fill in a
statically allocated buffer of bytes:
```asm
; --- <snip> ---
_start:
    call open           ; rax = fd of current directory
    call getdents64     ; dirents buffer filled
    call exit

getdents64:
    mov rdi, rax        ; rax = fd, move to rdi as argument to syscall
    mov rax, 217        ; getdents64 syscall
    mov rsi, dirents    ; pointer to buffer
    mov rdx, 1024       ; size of buffer
    syscall
    ret
; --- <snip> ---
section .bss
    dirents: resb 1024  ; allocate 1024 bytes
```
Now we basically have the directory listing in the memory at `dirents`, and the
number of bytes written in the `rax` register. All that's left is to write code
to loop through it and print the names of the directory entries.

### Printing a dirent
Printing a directory entry given a pointer to a `dirent` struct is pretty
straightforward with a couple short subroutines:
```asm
; --- <snip> ---
; input: rax = pointer to dirent
printdent:
    push rax
    add rax, dirent64.name
    call println        ; rax = pointer to null-terminated string
    pop rax
    ret

; input: rax = pointer to null-terminated string
println:
    call print
    push rax
    mov rax, newline
    call print
    pop rax
    ret

; input: rax = pointer to null-terminated string
print:
    call strlen         ; rdx = length of string
    call write
    ret
    
; input:  rax = pointer to null-terminated string
; output: rdx = length of string
strlen:
    xor rdx, rdx        ; len = 0
    strlen_loop:
        cmp byte [rax + rdx], 0
        je strlen_done
        inc rdx         ; len++
        jmp strlen_loop
    strlen_done:
    ret
    
; input: rax = pointer to null-terminated string
;        rdx = string length
write:
    mov rsi, rax
    mov rax, 1          ; write syscall
    mov rdi, 1          ; stdout file descriptor
    syscall
    ret

section .rodata
    newline: db 10, 0
; --- <snip> ---
```
Phew. Nothing too complicated though. Given a pointer to a dirent, it prints the
string at the correct offset in the struct, followed by a newline. The most
complicated piece of code here is `strlen`, which is still very simple.

### Loop!
Looping is achieved by simply incrementing a pointer to the start of the list by
`dirent64.reclen` for each dirent until the pointer has been incremented more
than the number of bytes written by the sycall. Recall that `getdents64` returns
the number of bytes written in the `rax` register. So with some simple pointer
arithmetic and comparisons, we can loop through all dirents in the list!
```asm
; --- <snip> ---
_start:
    call open
    call getdents64

    mov rbx, rax        ; bytes written
    mov rax, dirents    ; pointer to buffer
    call printdents
    call exit

; input: rax = pointer to dirents buffer
;        rbx = bytes written by syscall
printdents:
    add rbx, rax        ; pointer to end of buffer
    printdents_loop:    ; print while rax < rbx
        call printdent
        add ax, word [rax + dirent64.reclen] ; next dirent
        cmp rax, rbx
        jl printdents_loop
    ret
; --- <snip> ---
```
And believe it or not, that's the whole thing! It should be noted that it is
very hacky and not robust in the slightest for directories with a large number
of entries, but for just listing a couple of files in the current directory,
that's all you need! Assembling and linking this as `ls.asm` in an otherwise
empty directory, then running the resulting binary, will look something like
this:
```
$ nasm -felf64 ls.asm
$ ld ls.o -o ls
$ ./ls
ls.o
ls
.
ls.asm
..
```
The order might not be what you expect, but that's all you get with a hacky
implementation like this one ;).

Now, after running `strip` on the resulting binary, we can get the final size of
it:
```
$ strip ./ls
$ ls -lh ./ls
-rwxrwxr-x 1 martin martin 8,4K nov 30 19:50 ./ls
```
Now 8.4K might seem awfully big for such a small hand-written assembly program,
bust most of that is just padding to make sure the sections are aligned at 4K
page boundaries. We can get the actual size of the program using `size`:
```
$ size ./ls
   text	  data	   bss	   dec	   hex	filename
    191	     0	  1028	  1219	   4c3	./ls
```
We can see that it's only 191 bytes of code! That's pretty tiny. Still, the 8.4K
is well below the maximum of 32K given in the CTF problem. Oh, yeah! That's what
we were doing this for! Let's confirm that it's statically linked:
```
$ file ./ls
./ls: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```
Looks good! Here's the output after uploading it to the sandbox:
```
payload
..
.
```
Welp. That's it. The only file is the payload itself. And we can't access the
parent directories because the sandbox made sure of it when `chroot`-ing to this
temporary directory. At this point I gave up and moved on to other problems in
the CTF, because I had lost too much time and sleep over this one.

## Takeaways
The solution is probably not to hand-write assembly. If you spend a lot of time
on a difficult solution, that might be a sign that there's an easier way to do
it. And there is! The login binary never closed the file descriptor for
`userList.txt`. So all the payload had to do was use `lseek` to rewind the file
descriptor to the beginning, read the contents again, and spit it out using a
`write` syscall. All this can be achieved in C if you _do not use GCC_. I never
considered using a different compiler for some reason, but in one writeup I saw,
they wrote the C program I just described and mentioned that if you compile it
with `musl-cc`, the resulting elf will be less than 32K, and will contain enough
functionality to do everything in C, no assembly required.

Oh well, you live and you learn.

Here's the full listing for the crude implementation of `ls(1)`:

{{< labelled-highlight lang="asm" filename="ls.asm" >}}
global _start
section .text

struc dirent64
    .inode:  resb 8
    .offset: resb 8
    .reclen: resb 2
    .type:   resb 1
    .name:   resb 0
endstruc

_start:
    call open           ; rax = fd of current directory
    call getdents64     ; dirents buffer filled

    mov rbx, rax        ; bytes written
    mov rax, dirents    ; pointer to buffer
    call printdents
    call exit

; input: rax = pointer to dirents buffer
;        rbx = bytes written by syscall
printdents:
    add rbx, rax        ; pointer to end of buffer
    printdents_loop:    ; print while rax < rbx
        call printdent
        add ax, word [rax + dirent64.reclen] ; next dirent
        cmp rax, rbx
        jl printdents_loop
    ret

getdents64:
    mov rdi, rax        ; rax = fd, move to rdi as argument to syscall
    mov rax, 217        ; getdents64 syscall
    mov rsi, dirents    ; pointer to buffer
    mov rdx, 1024       ; size of buffer
    syscall
    ret

open:
    mov rax, 2          ; open syscall
    mov rdi, cwd        ; path to open (".")
    mov rsi, 0          ; O_RDONLY
    syscall
    ret

exit:
    mov rax, 60         ; exit syscall
    xor rdi, rdi        ; return value (0)
    syscall

; input: rax = pointer to dirent
printdent:
    push rax
    add rax, dirent64.name
    call println        ; rax = pointer to null-terminated string
    pop rax
    ret

; input: rax = pointer to null-terminated string
println:
    call print
    push rax
    mov rax, newline
    call print
    pop rax
    ret

; input: rax = pointer to null-terminated string
print:
    call strlen         ; rdx = length of string
    call write
    ret
    
; input:  rax = pointer to null-terminated string
; output: rdx = length of string
strlen:
    xor rdx, rdx        ; len = 0
    strlen_loop:
        cmp byte [rax + rdx], 0
        je strlen_done
        inc rdx         ; len++
        jmp strlen_loop
    strlen_done:
    ret
    
; input: rax = pointer to null-terminated string
;        rdx = string length
write:
    mov rsi, rax
    mov rax, 1          ; write syscall
    mov rdi, 1          ; stdout file descriptor
    syscall
    ret

section .rodata
    newline: db 10, 0
    cwd: db ".", 0
section .bss
    dirents: resb 1024  ; allocate 1024 bytes
{{</ labelled-highlight >}}
