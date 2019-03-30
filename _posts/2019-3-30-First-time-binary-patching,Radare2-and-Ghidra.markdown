---
title: First time binary patching,Radare2 and Ghidra.
tags: CTF binary-patching assembly
---
Today I decided to do another challenge on picoCTF2018, called be-quick-or-be-dead-1. A reverse engineering challenge with a description of,

"You find this when searching for some music, which leads you to be-quick-or-be-dead-1. Can you run it fast enough?"
the hint being "What will the key finally be?"

The hint didn't give me much to go on nor the description. I decided to boot up my docker container and run the program locally. not much output came out all I received was this.

(image)

I tried giving input before ruining it and tried typing something while it ran, but nothing happen. Based on what the hint said, maybe the key will be given or I will get something if I run it a bunch of times quickly. So I made a simple script that I thought would get something actionable.

However it showed nothing different. So since it was categorized as a Reverse engineering challenge I decided to decompile with objdump, which displayed some interesting results and showed it wasn't as simple as it seems. But my experience with assembly is very limited so I couldn't understand too much of it. I then decided to see the result as it runs with gdb and get a better understand on what happens during run time. Low and behold it just spit out the flag when I ran it inside gdb.

[image]()

Again I found the solution without understanding how I got there. I intended this post to be a proper write-up but instead I will use it as an opportunity to learn a little bit of assembly and binary patching. During Bsides2019 event I heard of a very interesting open source tool [Ghidra](https://ghidra-sre.org/) so, I will jump on the bandwagon and get to learning this interesting tool.

so upon opening this program up the import results summery showed that it was compiled with gcc which means its most likely a objective C program. Ghidra shows a C version of the assembly code, so lets use that. The best place would be to start at the main() function and go from there.

```c
undefined8 main(void)

{
  header();
  set_timer();
  get_key();
  print_flag();
  return 0;
}
```
So lets go through each one and see what each do.

header() function:

```c
void header(void)

{
  uint local_c;

  puts("Be Quick Or Be Dead 1");
  local_c = 0;
  while (local_c < 0x15) {
    putchar(0x3d);
    local_c = local_c + 1;
  }
  puts("\n");
  return;
}
```
Creates a unsigned integer "loacal_c". Prints out "Be Quick Or Be Dead 1". local_c equals to 0 then, a While loop while local_c is less than 0x15(21) and prints out 0x3d('=') then increments local_c.
after that it returns to main (Not too interesting of a function but I want to go through every thing)

set_timer()
```c
void set_timer(void)

{
  __sighandler_t pVar1;

  pVar1 = __sysv_signal(0xe,alarm_handler);
  if (pVar1 == (__sighandler_t)0xffffffffffffffff) {
    printf(
           "\n\nSomething went terribly wrong. \nPlease contact the admins with\"be-quick-or-be-dead-1.c:%d\".\n"
           ,0x3b);
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  alarm(1);
  return;
}
```

This is a bit more interesting, as the name says it sets a timer for something. lets break down its parts to see how its doing what its doing.
after googling __sighandler_t it appears to be from a GNU C library from this [website](http://www.gnu.org/software/libc/manual/html_node/Basic-Signal-Handling.html) it is a "Basic signal Handling" function. This particular datatype is a type of signal handler that takes one integer as a signal ID and returns 'void'.

Usage is demonstrated like this.
 ```c
Function: sighandler_t signal (int signum, sighandler_t action)
 ```

__sysv_signal is the binary's version of signal() which is what is used in source code and demonstrated above.

The 'signum' is a numerical code that indicates which signal you want to control. While the 'action' specifies the action to be used for the signal define by the 'signum'.

The signum used is 0xe which must represents SIGALRM, from the discerption given in documentation says it is used by the alarm() function which is present near the return statement.

The action is alarm_handler which the source code for it is this.

```c
void alarm_handler(void)

{
  puts("You need a faster machine. Bye bye.");
                    /* WARNING: Subroutine does not return */
  exit(0);
}

```
The if statement is just a error checking for picoctf people so I think it can be ignored.

The alarm function is set in seconds. So, in one second the 'action_handler' is used and it displays the message and stops the program. which is why the next function down below doesn't ever get to finish.

```c
void get_key(void)

{
  puts("Calculating key...");
  key = calculate_key();
  puts("Done calculating key");
  return;
}
```

 After looking at the other functions the calculate_key() does a pointless do while loop then sets a key to some value. The next function in main does gets the flag by decrypting it with the key and then prints it out. All this is hard to do in under a second.

This explains why the flag was picoCTF{why_bother_doing_unnecessary_computation_d0c6aace}. In any case with ghidra I tried to remove the set_timer function or the alarm by replacing it with 'NOP'. But it was very finicky and didn't work, seemed other people struggled with this as well. In the end I leaned and used radare2. A program I have seen before but it was a little too intimidating to use but, this [tutorial](https://scriptdotsh.com/index.php/2018/08/13/reverse-engineering-patching-binaries-with-radare2-arm-aarch64/) helped me out a lot and I got a feel for this incredible tool.

Here is how the patching looked.

[image](/assets\img\Posts\First-time-binaery-1.PNG)
[image](/assets\img\Posts\First-time-binaery-2.PNG)

Once I applied the NOP command I got a lot of invalid input bellow it in the disassembler, which I just continued to replace with 'NOP's until it went away.

[image](/assets\img\Posts\First-time-binaery-3.PNG)
[image](/assets\img\Posts\First-time-binaery-4.PNG)

Then I received my flag through my first binary patch.
