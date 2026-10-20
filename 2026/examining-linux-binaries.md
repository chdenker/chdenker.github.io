# Examining Linux ELF binaries

### Date: 2026-09-28

Recently, I was curious about what the structure of Linux binaries looks like in detail.
The standard file format on Linux for executable files (and object files and shared libraries) is the
"Executable and Linkable Format" (ELF), and this article illustrates this format using an example.

## From C to Assembly language

As a simple example, consider the following C program:

```c
int linsearch(int *arr, int n, int k)
{
    int i;
    for (i = 0; i < n; i++) {
        if (arr[i] == k) {
            return i;
        }
    }
    return -1;
}

int main(void)
{
    int arr[] = { 23, 42, -1, 99, 10, 1337, -2 };
    int n = sizeof arr / sizeof *arr;
    int k = 99;
    int i = linsearch(arr, n, k);
    return i;
}
```

We can compile and test the program as follows:

```bash
$ gcc -O0 \
    -fcf-protection=none \
    -fno-asynchronous-unwind-tables \
    -fno-ident \
    -fno-stack-protector \
    -o linear_search \
    linear_search.c
$ ./linear_search
$ echo $?
3
```

The `-f*` options I added to the `gcc` command remove some additional instructions, metadata and assembler directives that GCC adds for security reasons.
I use them here since they make it easier to match the binary we are going to look at to the assembly code.
You should probably not use them in production code unless you know what you are doing.

The `-S` option tells `gcc` to stop after the compilation stage.
The file `linear_search.s` then contains the generated assembly code.

```bash
$ gcc -S \
    -O0 \
    -fcf-protection=none \
    -fno-asynchronous-unwind-tables \
    -fno-ident \
    -fno-stack-protector \
    -o linear_search.s \
    linear_search.c
```

However if we try to assemble `linear_search.s` and link it into an object file, we get the following warning.

```bash
$ as -o linear_search.o linear_search.s
$ ld -o linear_search linear_search.o
ld: warning: cannot find entry symbol _start; defaulting to 0000000000401000
```

We resolve this by adding the following code at the start of the assembly file:

```asm
        .globl  _start
        .type   _start, @function
_start:
        call    main
        movl    %eax, %edi
        movl    $60, %eax
        syscall
```

This adds the entry symbol `_start` of ELF binaries and calls our `main` function.

This is the full assembly code (I am using gcc 13.3.0):

```asm
        .file   "linear_search.c"
        .text
        .globl  _start
        .type   _start, @function
_start:
        call    main
        movl    %eax, %edi
        movl    $60, %eax
        syscall

        .globl  linsearch
        .type   linsearch, @function
linsearch:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -24(%rbp)
        movl    %esi, -28(%rbp)
        movl    %edx, -32(%rbp)
        movl    $0, -4(%rbp)
        jmp     .L2
.L5:
        movl    -4(%rbp), %eax
        cltq
        leaq    0(,%rax,4), %rdx
        movq    -24(%rbp), %rax
        addq    %rdx, %rax
        movl    (%rax), %eax
        cmpl    %eax, -32(%rbp)
        jne     .L3
        movl    -4(%rbp), %eax
        jmp     .L4
.L3:
        addl    $1, -4(%rbp)
.L2:
        movl    -4(%rbp), %eax
        cmpl    -28(%rbp), %eax
        jl      .L5
        movl    $-1, %eax
.L4:
        popq    %rbp
        ret
        .size   linsearch, .-linsearch
        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $48, %rsp
        movl    $23, -48(%rbp)
        movl    $42, -44(%rbp)
        movl    $-1, -40(%rbp)
        movl    $99, -36(%rbp)
        movl    $10, -32(%rbp)
        movl    $1337, -28(%rbp)
        movl    $-2, -24(%rbp)
        movl    $7, -4(%rbp)
        movl    $99, -8(%rbp)
        movl    -8(%rbp), %edx
        movl    -4(%rbp), %ecx
        leaq    -48(%rbp), %rax
        movl    %ecx, %esi
        movq    %rax, %rdi
        call    linsearch
        movl    %eax, -12(%rbp)
        movl    -12(%rbp), %eax
        leave
        ret
        .size   main, .-main
        .section    .note.GNU-stack,"",@progbits
```

We assemble, link and execute the program as follows:

```bash
$ as -o linear_search.o linear_search.s
$ ld -o linear_search linear_search.o
$ ./linear_search 
$ echo $?
3
```

## The raw binary file

Let us look at the generated binary file:

```bash
$ ls -la linear_search
-rwxrwxr-x 1 chris chris 4896 Sep 28 21:02 linear_search
```

The file contains 4896 bytes and we can examine them using `hexdump`:

```bash
$ hexdump -C linear_search
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 3e 00 01 00 00 00  00 10 40 00 00 00 00 00  |..>.......@.....|
00000020  40 00 00 00 00 00 00 00  e0 11 00 00 00 00 00 00  |@...............|
00000030  00 00 00 00 40 00 38 00  03 00 40 00 05 00 04 00  |....@.8...@.....|
00000040  01 00 00 00 04 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 40 00 00 00 00 00  00 00 40 00 00 00 00 00  |..@.......@.....|
00000060  e8 00 00 00 00 00 00 00  e8 00 00 00 00 00 00 00  |................|
00000070  00 10 00 00 00 00 00 00  01 00 00 00 05 00 00 00  |................|
00000080  00 10 00 00 00 00 00 00  00 10 40 00 00 00 00 00  |..........@.....|
00000090  00 10 40 00 00 00 00 00  bb 00 00 00 00 00 00 00  |..@.............|
000000a0  bb 00 00 00 00 00 00 00  00 10 00 00 00 00 00 00  |................|
000000b0  51 e5 74 64 06 00 00 00  00 00 00 00 00 00 00 00  |Q.td............|
000000c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000000e0  10 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000  e8 53 00 00 00 89 c7 b8  3c 00 00 00 0f 05 55 48  |.S......<.....UH|
00001010  89 e5 48 89 7d e8 89 75  e4 89 55 e0 c7 45 fc 00  |..H.}..u..U..E..|
00001020  00 00 00 eb 24 8b 45 fc  48 98 48 8d 14 85 00 00  |....$.E.H.H.....|
00001030  00 00 48 8b 45 e8 48 01  d0 8b 00 39 45 e0 75 05  |..H.E.H....9E.u.|
00001040  8b 45 fc eb 11 83 45 fc  01 8b 45 fc 3b 45 e4 7c  |.E....E...E.;E.||
00001050  d4 b8 ff ff ff ff 5d c3  55 48 89 e5 48 83 ec 30  |......].UH..H..0|
00001060  c7 45 d0 17 00 00 00 c7  45 d4 2a 00 00 00 c7 45  |.E......E.*....E|
00001070  d8 ff ff ff ff c7 45 dc  63 00 00 00 c7 45 e0 0a  |......E.c....E..|
00001080  00 00 00 c7 45 e4 39 05  00 00 c7 45 e8 fe ff ff  |....E.9....E....|
00001090  ff c7 45 fc 07 00 00 00  c7 45 f8 63 00 00 00 8b  |..E......E.c....|
000010a0  55 f8 8b 4d fc 48 8d 45  d0 89 ce 48 89 c7 e8 5b  |U..M.H.E...H...[|
000010b0  ff ff ff 89 45 f4 8b 45  f4 c9 c3 00 00 00 00 00  |....E..E........|
000010c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000010d0  00 00 00 00 00 00 00 00  01 00 00 00 04 00 f1 ff  |................|
000010e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000010f0  16 00 00 00 12 00 01 00  00 10 40 00 00 00 00 00  |..........@.....|
00001100  00 00 00 00 00 00 00 00  11 00 00 00 10 00 01 00  |................|
00001110  00 20 40 00 00 00 00 00  00 00 00 00 00 00 00 00  |. @.............|
00001120  1d 00 00 00 12 00 01 00  58 10 40 00 00 00 00 00  |........X.@.....|
00001130  63 00 00 00 00 00 00 00  22 00 00 00 10 00 01 00  |c.......".......|
00001140  00 20 40 00 00 00 00 00  00 00 00 00 00 00 00 00  |. @.............|
00001150  29 00 00 00 10 00 01 00  00 20 40 00 00 00 00 00  |)........ @.....|
00001160  00 00 00 00 00 00 00 00  2e 00 00 00 12 00 01 00  |................|
00001170  0e 10 40 00 00 00 00 00  4a 00 00 00 00 00 00 00  |..@.....J.......|
00001180  00 6c 69 6e 65 61 72 5f  73 65 61 72 63 68 2e 63  |.linear_search.c|
00001190  00 5f 5f 62 73 73 5f 73  74 61 72 74 00 6d 61 69  |.__bss_start.mai|
000011a0  6e 00 5f 65 64 61 74 61  00 5f 65 6e 64 00 6c 69  |n._edata._end.li|
000011b0  6e 73 65 61 72 63 68 00  00 2e 73 79 6d 74 61 62  |nsearch...symtab|
000011c0  00 2e 73 74 72 74 61 62  00 2e 73 68 73 74 72 74  |..strtab..shstrt|
000011d0  61 62 00 2e 74 65 78 74  00 00 00 00 00 00 00 00  |ab..text........|
000011e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001220  1b 00 00 00 01 00 00 00  06 00 00 00 00 00 00 00  |................|
00001230  00 10 40 00 00 00 00 00  00 10 00 00 00 00 00 00  |..@.............|
00001240  bb 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001250  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001260  01 00 00 00 02 00 00 00  00 00 00 00 00 00 00 00  |................|
00001270  00 00 00 00 00 00 00 00  c0 10 00 00 00 00 00 00  |................|
00001280  c0 00 00 00 00 00 00 00  03 00 00 00 02 00 00 00  |................|
00001290  08 00 00 00 00 00 00 00  18 00 00 00 00 00 00 00  |................|
000012a0  09 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012b0  00 00 00 00 00 00 00 00  80 11 00 00 00 00 00 00  |................|
000012c0  38 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |8...............|
000012d0  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000012e0  11 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012f0  00 00 00 00 00 00 00 00  b8 11 00 00 00 00 00 00  |................|
00001300  21 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |!...............|
00001310  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001320
```

(The asterisk `*` means that the byte pattern above repeats.)

## Deciphering the sections

The first block until the second asterisk contains the ELF header. The command `readelf -h` makes the binary data readable:

```bash
$ readelf -h linear_search
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x401000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          4576 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         5
  Section header string table index: 4
```

Each byte of the first "magic" line has a fixed meaning:

| Byte index | Hex bytes | Meaning | `readelf` value |
| --- | --- | --- | --- |
| 0–3 | `7f 45 4c 46` | ELF magic number | `Magic: 7f 45 4c 46` |
| 4 | `02` | File class is ELF64 | `Class: ELF64` |
| 5 | `01` | Data encoding is little endian | `Data: 2's complement, little endian` |
| 6 | `01` | ELF version is 1 | `Version: 1 (current)` |
| 7 | `00` | OS/ABI is UNIX - System V | `OS/ABI: UNIX - System V` |
| 8 | `00` | ABI version is 0 | `ABI Version: 0` |
| 9–15 | `00 00 00 00 00 00 00` | Padding bytes | not displayed by `readelf` |

The most interesting part of the binary code is the `text` section which contains our actual program logic.
The command `objdump -d linear_search` is useful for examining this:

```bash
$ objdump -d linear_search

linear_search:     file format elf64-x86-64


Disassembly of section .text:

0000000000401000 <_start>:
  401000:    e8 53 00 00 00        call   401058 <main>
  401005:    89 c7                 mov    %eax,%edi
  401007:    b8 3c 00 00 00        mov    $0x3c,%eax
  40100c:    0f 05                 syscall

000000000040100e <linsearch>:
  40100e:    55                    push   %rbp
  40100f:    48 89 e5              mov    %rsp,%rbp
  401012:    48 89 7d e8           mov    %rdi,-0x18(%rbp)
  401016:    89 75 e4              mov    %esi,-0x1c(%rbp)
  401019:    89 55 e0              mov    %edx,-0x20(%rbp)
  40101c:    c7 45 fc 00 00 00 00  movl   $0x0,-0x4(%rbp)
  401023:    eb 24                 jmp    401049 <linsearch+0x3b>
  401025:    8b 45 fc              mov    -0x4(%rbp),%eax
  401028:    48 98                 cltq
  40102a:    48 8d 14 85 00 00 00  lea    0x0(,%rax,4),%rdx
  401031:    00 
  401032:    48 8b 45 e8           mov    -0x18(%rbp),%rax
  401036:    48 01 d0              add    %rdx,%rax
  401039:    8b 00                 mov    (%rax),%eax
  40103b:    39 45 e0              cmp    %eax,-0x20(%rbp)
  40103e:    75 05                 jne    401045 <linsearch+0x37>
  401040:    8b 45 fc              mov    -0x4(%rbp),%eax
  401043:    eb 11                 jmp    401056 <linsearch+0x48>
  401045:    83 45 fc 01           addl   $0x1,-0x4(%rbp)
  401049:    8b 45 fc              mov    -0x4(%rbp),%eax
  40104c:    3b 45 e4              cmp    -0x1c(%rbp),%eax
  40104f:    7c d4                 jl     401025 <linsearch+0x17>
  401051:    b8 ff ff ff ff        mov    $0xffffffff,%eax
  401056:    5d                    pop    %rbp
  401057:    c3                    ret

0000000000401058 <main>:
  401058:    55                    push   %rbp
  401059:    48 89 e5              mov    %rsp,%rbp
  40105c:    48 83 ec 30           sub    $0x30,%rsp
  401060:    c7 45 d0 17 00 00 00  movl   $0x17,-0x30(%rbp)
  401067:    c7 45 d4 2a 00 00 00  movl   $0x2a,-0x2c(%rbp)
  40106e:    c7 45 d8 ff ff ff ff  movl   $0xffffffff,-0x28(%rbp)
  401075:    c7 45 dc 63 00 00 00  movl   $0x63,-0x24(%rbp)
  40107c:    c7 45 e0 0a 00 00 00  movl   $0xa,-0x20(%rbp)
  401083:    c7 45 e4 39 05 00 00  movl   $0x539,-0x1c(%rbp)
  40108a:    c7 45 e8 fe ff ff ff  movl   $0xfffffffe,-0x18(%rbp)
  401091:    c7 45 fc 07 00 00 00  movl   $0x7,-0x4(%rbp)
  401098:    c7 45 f8 63 00 00 00  movl   $0x63,-0x8(%rbp)
  40109f:    8b 55 f8              mov    -0x8(%rbp),%edx
  4010a2:    8b 4d fc              mov    -0x4(%rbp),%ecx
  4010a5:    48 8d 45 d0           lea    -0x30(%rbp),%rax
  4010a9:    89 ce                 mov    %ecx,%esi
  4010ab:    48 89 c7              mov    %rax,%rdi
  4010ae:    e8 5b ff ff ff        call   40100e <linsearch>
  4010b3:    89 45 f4              mov    %eax,-0xc(%rbp)
  4010b6:    8b 45 f4              mov    -0xc(%rbp),%eax
  4010b9:    c9                    leave
  4010ba:    c3                    ret
```

It is important to note that the offsets at the beginning of the `objdump` output correspond to virtual addresses and not the byte offsets we have above in the `hexdump` output. Thus, the beginning of the text section at offset `00001000` of the `hexdump` output corresponds to the virtual address `401000`.

```bash
$ objdump -s -j .text linear_search

linear_search:     file format elf64-x86-64

Contents of section .text:
 401000 e8530000 0089c7b8 3c000000 0f055548  .S......<.....UH
 401010 89e54889 7de88975 e48955e0 c745fc00  ..H.}..u..U..E..
 401020 000000eb 248b45fc 4898488d 14850000  ....$.E.H.H.....
 401030 0000488b 45e84801 d08b0039 45e07505  ..H.E.H....9E.u.
 401040 8b45fceb 118345fc 018b45fc 3b45e47c  .E....E...E.;E.|
 401050 d4b8ffff ffff5dc3 554889e5 4883ec30  ......].UH..H..0
 401060 c745d017 000000c7 45d42a00 0000c745  .E......E.*....E
 401070 d8ffffff ffc745dc 63000000 c745e00a  ......E.c....E..
 401080 000000c7 45e43905 0000c745 e8feffff  ....E.9....E....
 401090 ffc745fc 07000000 c745f863 0000008b  ..E......E.c....
 4010a0 55f88b4d fc488d45 d089ce48 89c7e85b  U..M.H.E...H...[
 4010b0 ffffff89 45f48b45 f4c9c3             ....E..E... 
```

Other interesting sections:

```bash
$ readelf -S linear_search
There are 5 section headers, starting at offset 0x11e0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000401000  00001000
       00000000000000bb  0000000000000000  AX       0     0     1
  [ 2] .symtab           SYMTAB           0000000000000000  000010c0
       00000000000000c0  0000000000000018           3     2     8
  [ 3] .strtab           STRTAB           0000000000000000  00001180
       0000000000000038  0000000000000000           0     0     1
  [ 4] .shstrtab         STRTAB           0000000000000000  000011b8
       0000000000000021  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

We can list the symbols stored in the program as follows:

```bash
$ readelf --symbols linear_search

Symbol table '.symtab' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS linear_search.c
     2: 0000000000401000     0 FUNC    GLOBAL DEFAULT    1 _start
     3: 0000000000402000     0 NOTYPE  GLOBAL DEFAULT    1 __bss_start
     4: 0000000000401058    99 FUNC    GLOBAL DEFAULT    1 main
     5: 0000000000402000     0 NOTYPE  GLOBAL DEFAULT    1 _edata
     6: 0000000000402000     0 NOTYPE  GLOBAL DEFAULT    1 _end
     7: 000000000040100e    74 FUNC    GLOBAL DEFAULT    1 linsearch
```

The information displayed by `readelf --symbols` is stored in the `.symtab` section (except for the symbol names):

```bash
$ readelf -x .symtab linear_search

Hex dump of section '.symtab':
  0x00000000 00000000 00000000 00000000 00000000 ................
  0x00000010 00000000 00000000 01000000 0400f1ff ................
  0x00000020 00000000 00000000 00000000 00000000 ................
  0x00000030 16000000 12000100 00104000 00000000 ..........@.....
  0x00000040 00000000 00000000 11000000 10000100 ................
  0x00000050 00204000 00000000 00000000 00000000 . @.............
  0x00000060 1d000000 12000100 58104000 00000000 ........X.@.....
  0x00000070 63000000 00000000 22000000 10000100 c.......".......
  0x00000080 00204000 00000000 00000000 00000000 . @.............
  0x00000090 29000000 10000100 00204000 00000000 )........ @.....
  0x000000a0 00000000 00000000 2e000000 12000100 ................
  0x000000b0 0e104000 00000000 4a000000 00000000 ..@.....J.......
```

The section `.strtab` stores the names of the symbols. The symbol table itself just contains numeric offsets into `.strtab`, i.e. `.strtab + offset` is the location of the name.

```bash
$ readelf -x .strtab linear_search

Hex dump of section '.strtab':
  0x00000000 006c696e 6561725f 73656172 63682e63 .linear_search.c
  0x00000010 005f5f62 73735f73 74617274 006d6169 .__bss_start.mai
  0x00000020 6e005f65 64617461 005f656e 64006c69 n._edata._end.li
  0x00000030 6e736561 72636800                   nsearch.
```

The section `.shstrtab` contains a string table for the section headers.

```bash
$ readelf -x .shstrtab linear_search

Hex dump of section '.shstrtab':
  0x00000000 002e7379 6d746162 002e7374 72746162 ..symtab..strtab
  0x00000010 002e7368 73747274 6162002e 74657874 ..shstrtab..text
  0x00000020 00                                  .
```

The rest of the binary contains the ELF section header table. Note that the output of `readelf -h linear_search` contained the line:

```bash
Start of section headers:          4576 (bytes into file)
```

The decimal value 4576 corresponds to the hexadecimal value `0x11e0`, which matches the remaining portion of the binary:

```bash
000011e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001220  1b 00 00 00 01 00 00 00  06 00 00 00 00 00 00 00  |................|
00001230  00 10 40 00 00 00 00 00  00 10 00 00 00 00 00 00  |..@.............|
00001240  bb 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001250  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001260  01 00 00 00 02 00 00 00  00 00 00 00 00 00 00 00  |................|
00001270  00 00 00 00 00 00 00 00  c0 10 00 00 00 00 00 00  |................|
00001280  c0 00 00 00 00 00 00 00  03 00 00 00 02 00 00 00  |................|
00001290  08 00 00 00 00 00 00 00  18 00 00 00 00 00 00 00  |................|
000012a0  09 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012b0  00 00 00 00 00 00 00 00  80 11 00 00 00 00 00 00  |................|
000012c0  38 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |8...............|
000012d0  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000012e0  11 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012f0  00 00 00 00 00 00 00 00  b8 11 00 00 00 00 00 00  |................|
00001300  21 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |!...............|
00001310  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001320
```

This block contains the section headers of the sections listed in the `readelf -S` output.
Each section header contains information about one section, such as the section name (stored as an index to the actual name which is stored in the `.shstrtab` section),
section type, virtual address in memory, file offset of the section data, the length and more.

## The annotated binary file

Finally, we can understand what we are seeing in the hexdump.

### ELF header

```text
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 3e 00 01 00 00 00  00 10 40 00 00 00 00 00  |..>.......@.....|
00000020  40 00 00 00 00 00 00 00  e0 11 00 00 00 00 00 00  |@...............|
00000030  00 00 00 00 40 00 38 00  03 00 40 00 05 00 04 00  |....@.8...@.....|
00000040  01 00 00 00 04 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 40 00 00 00 00 00  00 00 40 00 00 00 00 00  |..@.......@.....|
00000060  e8 00 00 00 00 00 00 00  e8 00 00 00 00 00 00 00  |................|
00000070  00 10 00 00 00 00 00 00  01 00 00 00 05 00 00 00  |................|
00000080  00 10 00 00 00 00 00 00  00 10 40 00 00 00 00 00  |..........@.....|
00000090  00 10 40 00 00 00 00 00  bb 00 00 00 00 00 00 00  |..@.............|
000000a0  bb 00 00 00 00 00 00 00  00 10 00 00 00 00 00 00  |................|
000000b0  51 e5 74 64 06 00 00 00  00 00 00 00 00 00 00 00  |Q.td............|
*
000000e0  10 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
```

### .text section

```text
00001000  e8 53 00 00 00 89 c7 b8  3c 00 00 00 0f 05 55 48  |.S......<.....UH|
00001010  89 e5 48 89 7d e8 89 75  e4 89 55 e0 c7 45 fc 00  |..H.}..u..U..E..|
00001020  00 00 00 eb 24 8b 45 fc  48 98 48 8d 14 85 00 00  |....$.E.H.H.....|
00001030  00 00 48 8b 45 e8 48 01  d0 8b 00 39 45 e0 75 05  |..H.E.H....9E.u.|
00001040  8b 45 fc eb 11 83 45 fc  01 8b 45 fc 3b 45 e4 7c  |.E....E...E.;E.||
00001050  d4 b8 ff ff ff ff 5d c3  55 48 89 e5 48 83 ec 30  |......].UH..H..0|
00001060  c7 45 d0 17 00 00 00 c7  45 d4 2a 00 00 00 c7 45  |.E......E.*....E|
00001070  d8 ff ff ff ff c7 45 dc  63 00 00 00 c7 45 e0 0a  |......E.c....E..|
00001080  00 00 00 c7 45 e4 39 05  00 00 c7 45 e8 fe ff ff  |....E.9....E....|
00001090  ff c7 45 fc 07 00 00 00  c7 45 f8 63 00 00 00 8b  |..E......E.c....|
000010a0  55 f8 8b 4d fc 48 8d 45  d0 89 ce 48 89 c7 e8 5b  |U..M.H.E...H...[|
000010b0  ff ff ff 89 45 f4 8b 45  f4 c9 c3 00 00 00 00 00  |....E..E........|
```

### .symtab section

```text
000010c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000010d0  00 00 00 00 00 00 00 00  01 00 00 00 04 00 f1 ff  |................|
000010e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000010f0  16 00 00 00 12 00 01 00  00 10 40 00 00 00 00 00  |..........@.....|
00001100  00 00 00 00 00 00 00 00  11 00 00 00 10 00 01 00  |................|
00001110  00 20 40 00 00 00 00 00  00 00 00 00 00 00 00 00  |. @.............|
00001120  1d 00 00 00 12 00 01 00  58 10 40 00 00 00 00 00  |........X.@.....|
00001130  63 00 00 00 00 00 00 00  22 00 00 00 10 00 01 00  |c.......".......|
00001140  00 20 40 00 00 00 00 00  00 00 00 00 00 00 00 00  |. @.............|
00001150  29 00 00 00 10 00 01 00  00 20 40 00 00 00 00 00  |)........ @.....|
00001160  00 00 00 00 00 00 00 00  2e 00 00 00 12 00 01 00  |................|
00001170  0e 10 40 00 00 00 00 00  4a 00 00 00 00 00 00 00  |..@.....J.......|
```

### .strtab section

```text
00001180  00 6c 69 6e 65 61 72 5f  73 65 61 72 63 68 2e 63  |.linear_search.c|
00001190  00 5f 5f 62 73 73 5f 73  74 61 72 74 00 6d 61 69  |.__bss_start.mai|
000011a0  6e 00 5f 65 64 61 74 61  00 5f 65 6e 64 00 6c 69  |n._edata._end.li|
000011b0  6e 73 65 61 72 63 68 00  00 2e 73 79 6d 74 61 62  |nsearch...symtab|
000011c0  00 2e 73 74 72 74 61 62  00 2e 73 68 73 74 72 74  |..strtab..shstrt|
000011d0  61 62 00 2e 74 65 78 74  00 00 00 00 00 00 00 00  |ab..text........|
```

### .shstrtab section

```text
000011b8  00 2e 73 79 6d 74 61 62  00 2e 73 74 72 74 61 62  |..symtab..strtab|
000011c8  00 2e 73 68 73 74 72 74  61 62 00 2e 74 65 78 74  |..shstrtab..text|
```

### ELF section header table

```text
000011e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001220  1b 00 00 00 01 00 00 00  06 00 00 00 00 00 00 00  |................|
00001230  00 10 40 00 00 00 00 00  00 10 00 00 00 00 00 00  |..@.............|
00001240  bb 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001250  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001260  01 00 00 00 02 00 00 00  00 00 00 00 00 00 00 00  |................|
00001270  00 00 00 00 00 00 00 00  c0 10 00 00 00 00 00 00  |................|
00001280  c0 00 00 00 00 00 00 00  03 00 00 00 02 00 00 00  |................|
00001290  08 00 00 00 00 00 00 00  18 00 00 00 00 00 00 00  |................|
000012a0  09 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012b0  00 00 00 00 00 00 00 00  80 11 00 00 00 00 00 00  |................|
000012c0  38 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |8...............|
000012d0  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000012e0  11 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
000012f0  00 00 00 00 00 00 00 00  b8 11 00 00 00 00 00 00  |................|
00001300  21 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |!...............|
00001310  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001320
```

## Final notes

This is the ELF binary of a very simple C program and more complex programs will contain additional sections.
Most notably, the data segment `.data` for initialized statically allocated variables (global variables and static local variables) and the
`.bss` ("block started by symbol" or "block starting symbol") section for uninitialized statically allocated variables.
The read-only data segment `.rodata` is where static constants (and string constants) end up.

## Further reading

1. [Wikipedia: "Executable and Linkable Format"](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
2. [Linux elf(5) man page](https://man7.org/linux/man-pages/man5/elf.5.html)
