---
title:  "Hello-World!-Hacker101-CTF"
date:   2019-05-29 20:0:0 +1100
tags: CTF writeup
---
So finally have some time to do a CTF. This time I am going to do a challenge on [Hacker101](https://www.hacker101.com/) called Hello World!. lets get started.

First thing we are given is this.

![image](/assets\img\Posts\Hacker101 Hello world\img1.PNG)

When you give it a input it displays it below. The source code of the website showed nothing interesting so I downloaded the binary file and open it up in Ghidra. using the C interpretation functionality I got this output.

The main function,

```
undefined8 main(void)

{
  char local_28 [32];

  memset(local_28,0,0x20);
  read_all_stdin(local_28);
  if (local_28[0] == 0) {
    puts("What is your name?");
  }
  else {
    printf("Hello %s!\n",local_28);
  }
  return 0;
}
```

It defines a character buffer to the length of 32 bytes. initializes the elements to 0 with ```memset()``` then reads from stdin and displays the result to the webpage. Lets look at the ```read_all_stdin()``` function.

```
void read_all_stdin(long lParm1)

{
  int iVar1;
  int local_c;

  local_c = 0;
  while( true ) {
    iVar1 = fgetc(stdin);
    if (iVar1 == -1) break;
    *(undefined *)(lParm1 + (long)local_c) = (char)iVar1;
    local_c = local_c + 1;
  }
  return;
}
```

It initializes some variables and then takes each character from stdin with ```fgetc()``` and stores it in a variable. Until it receives a '-1' it will place that character in the array 'lParm1' our local_28 from before.

The function that will give us our flag is this,

```
void print_flags(void)

{
  char *__s;

  __s = getenv("FLAGS");
  puts(__s);
                    /* WARNING: Subroutine does not return */
  exit(0);
}
```

So there isn't any protection around the buffer as the ```printf()``` function in main will continue printing from the stack until it receives a null terminator, this gives us the opportunity for a buffer overflow which we can change the return address to point to the ```print_flags()``` function. If we give the input 31 A's the out put is fine however, if its given 32 A's it shows an interesting output.  

![image](/assets\img\Posts\Hacker101 Hello world\img2.PNG)

It displays some random characters after our 32 A's. If we increment the number of A's we see that the random characters start disappearing then at 40 A's more random characters appear, then a 41 A's we get a segfault. because we overwritten the null terminator of local_28 the ```printf()``` will keep reading the stack until it hits a null terminator hence those extra characters are just bytes from the stack that are not null.

To better show this if you add a %00 (a null byte) in-between the input, it wont print anything after it (put it directly in the url so that it wont take the % as a literal string by the form tag)

![image](/assets\img\Posts\Hacker101 Hello world\img3.PNG)

When we use 41 A's we get a segfault because the buffer has overflown and overwrote the return address, when the return function try's to read the new address it cant find it anywhere in memory so it breaks the program but, if we give it a address that is valid we can make the program do whatever we want, like print a flag.

By using the readelf command in kali linux (append option -s), we can search for the address of the ```print_flags()``` function we want to execute.

![image](/assets\img\Posts\Hacker101 Hello world\img4.PNG)

So the address is 0x00 00 00 00 00 40 06 ee (because the program uses little endian it will be reverse 0x ee 06 40 00 00 00 00 00) so add the 40 A's to overflow the buffer then add in the payload which is the function inside the program we want to run. (make sure to add directly in the url so that it keeps the % signs as they are) it will look something like this,

![image](/assets\img\Posts\Hacker101 Hello world\img5.PNG)

And that gets us our flag! thanks for reading hopefully it was helpful.

Conclusion:

This CTF was fun but had a lot challenges, luckily with help my mate Ed, I was able to get to the end.

First major problem was, use to doing buffer overflow challenges on a shell server or with a standalone program, it was very interesting to deal with this problem on a website, so it took a while of experimenting for me to figure out how to put hex as a input. I figured it out because any special character like, 'backslash' is represented by the hex %5c. so by using the % sign I could indicate my hex values to the stdin.

Another problem was understanding the code, I was trying to find a different problem with the code like a miss spelling or bad function. usually in buffer overflow challenges they use strcpy() or other well know exploitable functions. I tried to look for what was there rather than seeing what wasn't like, the fact there was nothing checking how much data was going into the buffer. There was a check to see if a character was equal to -1 then if would terminate the program but that doesn't provide sufficient security.

In general I struggled to deal with code that was presented differently, the way the characters was added to the buffer using pointers and written in a way that I was not use to, seemed unusual and weird so I thought that a problem had to be there. It was only when my friend Ed asked me questions about the problem and getting me to analysis it in a succinct way did I start to have some things clear up for me.

Another issue was general testing with gdb, there was problems with uses of privileged functions in a docker container that didnt have them activated. But I will be prepared next time.

All in all I've done better on this challenge and learned alot.
