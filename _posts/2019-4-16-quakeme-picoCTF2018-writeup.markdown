---
title: quakeme-picoCTF2018-writeup
tags: CTF binary-patching writeup
---

Here is a proper writeup of the quakeme problem from picoCTF2018. Lets get quacking...

you may want to run the program first but I wont bother for the purpose of this write up.
First decompile/ disassemble the program with any tool (I will be on the Ghidra bandwagon for the most part) and locate the main function.

![image](/assets\img\Quakeme-0.png)

Disassembling the do_magic() function gives this result.


```
void do_magic(void)

{
  char *__s;
  size_t sVar1;
  void *__s_00;
  int local_20;
  int local_1c;

  __s = (char *)read_input();
  sVar1 = strlen(__s);
  __s_00 = malloc(sVar1 + 1);
  if (__s_00 == (void *)0x0) {
    puts("malloc() returned NULL. Out of Memory\n");
                    /* WARNING: Subroutine does not return */
    exit(-1);
  }
  memset(__s_00,0,sVar1 + 1);
  local_20 = 0;
  local_1c = 0;
  while( true ) {
    if ((int)sVar1 <= local_1c) {
      return;
    }
    if (greetingMessage[local_1c] == (char)(__s[local_1c] ^ sekrutBuffer[local_1c])) {
      local_20 = local_20 + 1;
    }
    if (local_20 == 0x19) break;
    local_1c = local_1c + 1;
  }
  puts("You are winner!");
  return;
}
```

The function before the ```memset(__s_00,0,sVar1 + 1);``` is not of too much concern. It's setting up memory allocation based on the size of the string we give, preventing buffer overflows and memory corruption. The next section of the code is were the real "magic" happens.

The first if statement checks if the string "sVar1" is less then "local_1c" which is acting as our counter. If true it will exit the while loop.

The next if statement is comparing a character from the "greetingMessage" with a character that's from our input string, XORing with a character from "sekrutBuffer". all arrays are being indexed by the counter "local_1c". If true it increments "local_20".

Then it checks it "local_20" equals to 0x19 (25 in decimal). If true it breaks out of the program.

finally it increments our counter at the end.

No patching is required for this challenge, the flag can be derived manually. Because our input variable is being XORed with "sekrutBuffer" and "greetingMessage" is our output of that XOR, we can reverse it by XORing with the output. By getting the hexadecimal stored in each array via Ghidra, we can input these values to find our flag Here is the code I used that demonstrates this.

```
#include <stdio.h>

int main()
{
  //XOR by 's' first
  char ch = 'a';
  printf("%c\n",ch);
  ch = ch ^ 's';
  printf("%c\n",ch);

  //Reclaim 'a' by XORing with 's' again
  ch = ch ^ 's';
  printf("%c\n",ch);

  //c = a^b; a is our input string, b is sekrutBuffer, c is greetingMessage
  //a = c^b
  char b[25] = {0x29, 0x06, 0x16, 0x4f, 0x2b, 0x35, 0x30, 0x1e, 0x51, 0x1b, 0x5b, 0x14, 0x4b, 0x08, 0x5d, 0x2b, 0x5c, 0x10, 0x06, 0x06, 0x18, 0x45, 0x51, 0x00, 0x5d, 0x00};
  char c[25] = {0x59, 0x6f, 0x75, 0x20, 0x68, 0x61, 0x76, 0x65, 0x20, 0x6e, 0x6f, 0x77, 0x20, 0x65, 0x6e, 0x74, 0x65, 0x72, 0x65, 0x64, 0x20, 0x74, 0x68, 0x65, 0x20, 0x44};
  char a[25] = {};

  for (int i = 0; i < 25; i++){
    a[i] = c[i] ^ b[i];
    printf("%c", a[i]);
  }

  return 0;
}
```
This give us the flag picoCTF{qu4ckm3_9bcb819e}. YAY!

Conlusion:

So this was a unexpected challenge I was prepared to modify some binaries but, instead I got to just figure it out with out any fuss. But like always I made some silly mistakes, for quite awhile doing this challenge I was confused on reversing the ```__s[local_1c] ^ sekrutBuffer[local_1c]``` because I miss took the '^' symbol as a power sign not a binary operation.

However, I am glad that I stuck it though this challenge as that simple silly mistake, made a massive headache. I nearly gave in to looking at writeup to see where I was going wrong but, instead I looked on forums and received advice instead of answers that help me relies my mistake.

Its a very common problem I keep seeing myself get stuck in (especially writing these posts), were I struggle to stay focus and committed to a challenge like this. I get frustrated easily when I don't know how something works or how to do something but, at less I am aware and improving on this. Which will most defiantly aid me in my development as a hacker.

That's it, I had fun doing and writing up this challenge. hopefully it helps someone.
Thanks for reading.
