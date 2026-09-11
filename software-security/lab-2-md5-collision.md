---
title: Laboration 2 - MD5 Collision
---

[Lab description](https://seedsecuritylabs.org/Labs_16.04/PDF/Crypto_MD5_Collision.pdf)

## Environment & tools
The provided SEED lab environment was used, which is a preconfigured virtual machine image running a 32-bit version of Ubuntu 16.04. It is being run in VirtualBox on a 64-bit Linux host.

For performing MD5 collision attacks, the `md5collgen` tool as provided by the SEED Lab environment will be used.

## Task 1: Generating Two different Files with the Same MD5 Hash
Let's start off by generating a very simple collision. We write something into a file called `prefix.txt`:

```bash
$ echo "cuddles" > prefix.txt
```

Which is then used as prefix to generate two files using `md5collgen`:

```bash
$ md5collgen -p prefix.txt -o out{1..2}.bin
```

Checking the state of the files with `diff` shows that they do in fact differ, but calculating their MD5 sum shows that they are the same. A hash collision has occurred.

```bash
$ diff out{1..2}.bin
Binary files out1.bin and out2.bin differ
$ md5sum out{1..2}.bin
6949659241af134d02a7ce4d4e4fd84b  out1.bin
6949659241af134d02a7ce4d4e4fd84b  out2.bin
```

(Question 1) Since the prefix is not a multiple of 64, inspecting the file with a hex editor shows that the program has padded out the prefix to the nearest multiple of 64, meaning beyond the "cuddles" and newline, there are null bytes up to byte 64, where the MD5 collision generator will generate random bytes to create a collision.

(Question 2) Trying it on another prefix that is exactly 64 bytes in length, a bunch of A's repeated, also creates another pair of hash colliding files.

```bash
$ cat 64coll.txt | wc -c
64
$ md5collgen -p 64coll.txt -o 64coll{1..2}.bin
[...]
$ diff 64coll{1..2}.bin
Binary files 64coll.bin and 64coll.bin differ
$ md5sum 64coll{1..2}.bin
3a67e6eb6d4289b8a6f255f3fe7db870  64coll1.bin
3a67e6eb6d4289b8a6f255f3fe7db870  64coll2.bin
```

Inspecting it with a hex editor shows that this time there is no padding - right when the A's stop, the random bytes for creating a hash collision begin. There is clearly something special about 64 bytes in MD5.

(Question 3) Looking at the difference between the random bytes in the two generated files, a lot of the bytes are actually the same, but only a few bytes have been changed in order to create a collision match. Screenshot of the two files opened in the program `hexdiff`, highlighting bytes that are different:

{% include image.html
	url="/software-security/hexdiff.webp"
	max_width=600 %}

## Task 2: Understanding MD5's Property
As the task description explains, and as was demonstrated above, the MD5 hashing algorithm works in 64 byte blocks. The core of the algorithm is that for each block, a hash (a so-called Intermediate Hash Value) of the previous block is combined with the current data block that gets hashed, until it reaches the end of the file and the final hash is returned.

This will mean that if you concatenate the same content onto two files with the same hash, then the resulting hashes will be the same for both files. This should also hold true for files that are different, but have colliding hashes. Let's experiment.

We once again have two pairs of differing files that both have the same hash, with an identical prefix and two 64 byte blocks that cause a collision.

```bash
$ diff out{1..2}.bin
Binary files out1.bin and out2.bin differ
$ md5sum out{1..2}.bin
2d9fbd00aaa74bead6618b593a9c168e  out1.bin
2d9fbd00aaa74bead6618b593a9c168e  out2.bin
```

Let's concatenate the same file onto both of these files, let's call the file `suffix.txt`:

```bash
$ echo "cuddles" > suffix.txt
$ cat out1.bin suffix.txt > out1_concat.bin
$ cat out2.bin suffix.txt > out2_concat.bin
$ md5sum out{1..2}_concat.bin
4dfefcdc8add1143da7a2acff695377f  out1_concat.bin
4dfefcdc8add1143da7a2acff695377f  out2_concat.bin
```

The hash differs from the original two files, but the hashes are still identical to eachother. This means that we can both control the prefix and the suffix, as long as they are the same and we have somewhere to place the 128 byte collision blocks.

## Task 3: Generating Two Executable Files with the Same MD5 Hash
Now that we can surgically insert the 128 byte collision blocks into a file with arbitrary (identical) prefixes and suffixes, let's try to create a hash collision with a simple C program.

This is the code for the program that will be used, as provided in the task description. The `xyz` array is 200 bytes in length and contains 200 bytes of 0x41 ("A"). The program itself will simply print these out to the terminal in hex representation.

```c
#include <stdio.h>

unsigned char xyz[200] = {
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	[...]
};

int main() {
	int i;
	for (int i = 0; i < 200; i++)
		printf("%x", xyz[i]);
	printf("\n");
}
```

Compiling it with GCC:

```bash
$ gcc coll.c -o coll
```

Looking at the compiled executable file in a hex editor shows that the static text data of the array starts at location `0x00001040`. In decimal this is 4160, which is also evenly divisible by 64!

To cut up the executable, the following commands were run:

```bash
$ head -c 4160 coll > prefix
$ tail -c +4287 coll > suffix
```

Then generating two colliding files using the prefix:

```bash
$ md5collgen -p prefix -o coll_{1..2}
```

And concatenating the suffixes:

```bash
$ cat coll_1 suffix > coll_exe1
$ cat coll_2 suffix > coll_exe2
```

Then diff'ing them shows they differ, while their MD5 hashes are the same.

```bash
$ diff coll_exe{1..2}
Binary files coll_exe1 and coll_exe2 differ
$ md5sum coll_exe{1..2}
4ed5b6c5d0d6f26bd174a62a811f0da6  coll_exe1
4ed5b6c5d0d6f26bd174a62a811f0da6  coll_exe2
$ chmod +x coll_exe{1..2}
```

And trying to run the resulting executables works fine without any segfaults, printing out the contents of the array in which the collision blocks were put.

```bash
$ ./coll_exe1
[...]
$ ./coll_exe2
[...]
```

## Task 4: Making the Two Programs Behave Differently
While the previous program pair would behave exactly the same, we can also craft two programs that share the same hash, but behave differently. In the example the task brings up, one program would be good and one would be evil and malicious. The good version would be sent to a certification authority for approval, and once it has been signed the MD5 hash would also apply to the evil version.

To create this the program would have two byte arrays that will be compared against eachother. If they are the same, then the good program codepath will be executed, else the bad program codepath will be executed. So the collision would be run and put in the first byte array, and then the second byte array would have the contents of one program's first array, in both programs. This way there will be one program where the arrays match and one where they do not match, while both having the same hash.

Writing actual malware is obviously out of scope of the laboration, so for demonstration it will just print "Good program! :)" and "Evil program! >:)" for respective program. Below is the code used, checking whether the arrays are equal using `memcmp`:

```c
#include <stdio.h>
#include <string.h>

unsigned char x[200] = {
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	[...]
};

unsigned char y[200] = {
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41,
	[...]
};

int main() {
	if (memcmp(x, y, 200) == 0) {
		printf("Good program! :)\n");
	} else {
		printf("Evil program! >:)\n");
	}
}
```

Compiling it with GCC:

```bash
$ gcc program.c -o program
```

Looking at the compiled executable file in a hex editor shows that the static text data for one of the arrays starts at location `0x00001040` (concidentally the same as in the previous program).

```bash
$ head -c 4160 program > prefix
$ tail -c +4287 program > suffix
```

Then generating two colliding files using the prefix:

```bash
$ md5collgen -p prefix -o coll_{1..2}
```

This generates the beginning of two colliding programs. If we were to just concatenate the suffix then both programs would become evil as the second array that's located in the suffix has not been modified yet. To make one of the programs good, we will need to put one of the program's first array contents into the second array that is in the suffix.

Let's extract it from `coll_1`:

```bash
$ tail -c 128 coll_1 > blob_p
```

Then cut up the suffix into two parts, before and after the `blob_p` file should go.

```bash
$ head -c 96 suffix > new_suf_1
$ tail -c +224 suffix > new_suf_3
```

Then between those, the colliding `blob_p` file will be concatenated, like such:

```bash
$ cat coll_1 new_suf_1 blob_p new_suf_3 > outprog_good
$ cat coll_2 new_suf_1 blob_p new_suf_3 > outprog_evil
```

Now a pair of colliding programs have been created. Trying to run each program shows that they behave differently:

```bash
$ ./outprog_good
Good program! :)
$ ./outprog_evil
Evil program! >:)
```

Of course, their MD5 hashes match:

```bash
$ md5sum outprof_{good,evil}
aa5e03fb6af0e04e310614e7217cd210  outprog_good
aa5e03fb6af0e04e310614e7217cd210  outprog_evil
```

So when sending the program for certification, `outprog_good` would be sent showing that it is a benign program. And once the certification for the `outprog_good` program's MD5 is approved, it will also apply to `outprog_evil` when they have the same MD5 hash. So then sending off `outprog_evil` to be used as e.g. a trojan horse would make it seem like it has been confirmed to be safe by the certification authority, when it's actually malware.

However it is worth noting that the evil codepath and the if condition to switch between the good and evil codepath is present in the good program, and could be discovered by the certification authority if they were to inspect the program at a deeper level with a disassembler.

A more elaborate collision generation would be needed to remove the evil code from the good program, or some more obfuscation to try to distract a nosy certification authority. But this task still demonstrates the ability to create two colliding programs with the same hash but with completely different behaviour when executed, which would end up being a disastrous scenario for the certification of executable files.
