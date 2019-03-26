---
title:  "Pseudo randomness and Integer Overflows"
date:   2019-03-24 20:0:0 +1100
tags: CTF PRNG integer-overflow
---

So finally my first actual blog post. YAY!! (ﾉ◕ヮ◕)ﾉ*:･ﾟ✧

I was doing a CTF challenge on [picoCTF2018](https://2018game.picoctf.com/game) called Roulette. The challenge's description was,
"This Online Roulette Service is in Beta. Can you find a way to win $1,000,000,000 and get the flag? Source."
the hint given was there were "two bugs to find".

>( side note ) The order of how I explain my findings will be the order I found them while I was doing this challenge, so there maybe some going back and fourth and seem a bit chaotic. But this is showing my process and all the ups and downs. it is not a structured writeup for someone to walkthrough. sorry in advance if its messy.

With any CTF challenge I struggle to stay committed to it, putting in the time and not give up by looking at writeups online. So this time I was committed to not taking any shortcuts.
I looked at the source code, thought the best place to start would be to look at where the user input is needed, where it goes and see what I can control.

here is a snippet of the main() function.

 ``` c
  long bet;
  long choice;
  while(cash > 0) {
      bet = get_bet();
      cash -= bet;
      choice = get_choice();
      puts("");

      play_roulette(choice, bet);

 ```
So two things it needs from user is the bet and what number they pick. I also take a quick look at what I need to consider to get the flag so here are the win conditions of the main function.

 ``` c
 if(cash > ONE_BILLION) {
  printf("*** Current Balance: $%lu ***\n", cash);
  if (wins >= HOTSTREAK) {
     puts("Wow, I can't believe you did it.. You deserve this flag!");
     print_flag();
     exit(0);
}	else {
puts("Wait a second... You're not even on a hotstreak! Get out of here cheater!");
exit(-1);
 ```
The (HOTSTREAK = 3) so I need to have won three games before I get one billion (becomes a interesting factor of the next bug), so this challenge seems to be messing with numbers stored and the way to win. First lets look at the get_bet() function.

``` c
long get_bet() {
  while(1) {
    puts("How much will you wager?");
    printf("Current Balance: $%lu \t Current Wins: %lu\n", cash, wins);
    long bet = get_long();
    if(bet <= cash) {
      return bet;
    } else {
      puts("You can't bet more than you have!");
    }
  }
}
```
Nothing seems breakable yet however to get users input it calls get_long() and stores it in a long variable called 'bet'. lets have a look a get_long().

``` c
long get_long() {
    printf("> ");
    uint64_t l = 0;
    char c = 0;
    while(!is_digit(c))
      c = getchar();
    while(is_digit(c)) {
      if(l >= LONG_MAX) {
	l = LONG_MAX;
	break;
      }
      l *= 10;
      l += c - '0';
      c = getchar();
    }
    while(c != '\n')
      c = getchar();
    return l;
}
```
Now we have a function that now makes a call to ANOTHER function which all it does is ensures each character given is indeed a number (take my word for it :p)
So it has a uint64_t (unsigned 8 bytes integer)'l' but it returns a long (signed 4 bytes integer) value. I thought that there must be a problem here to mess with, while it maybe be obvious to some of you but, at the time I didn't understand how I could mess with this. I tried giving a big value like 99999999999999999999 but it would be able handle it by taking in every number individually and add it one at a time. I tried putting in negative numbers being that the variable 'l' was unsigned but it didn't change anything.

So I then looked at the get_choice() function and it too used the get_long() function to receive users input. but there was still no way I found to abuse the problem that was defiantly here.

What happened eventually while I was doing this challenge, was I went onto picoctf's 'plaza' a discussion forum to discuss challenges and help each other. At fist I was a little frustrated with myself that I had to rely on this forum but no one gave out the answer, they gave out better hints and direction so I still sort of felt like I kept to my commitment to not taking short cuts. After this a user had gave hint to the other bug in this challenge looking at how the randomness is generated and that would lead onto winning rounds in the challenge.

So now I wasn't sure that I was even looking in the right place once I read that comment. during this time I left this section in pursuit of that bug. For the sake of consistence of this blog post we will skip ahead to the point after I found that bug (ill discuss the process of that later) and continue on how I eventually solved this problem.

The second bug gave me more insight in how to analyse a problem better. The first thing was to analyse the code more carefully and the second was ...

![knowledge](/assets\img\emoji\knowledge.gif)  
(brownie points to you if you remember this dead meme)

I did google and try to learn more about what I was dealing with before. But, I did not take my time with it. so the next day on this challenge I came with fresh eyes and hopped on google.

what is uint64_t datatype?: It is a unsigned (meaning the first bit of the number does not represent a positive or negative) 8 bytes (64 bits) integer with a range of 0 to
what is a long datatype?: a signed (first bit represents a positive or negative sign) 4 bytes (32 bits) integer with a range of -2,147,483,648 to 2,147,483,647
what is LONG_MAX?: the max value of long data type 2,147,483,647.
what is the process of get_long() function?:
>1) Gets the next character with getchar() in the given string input and stores it in variable 'c'. then checks 'c' if its a digit with is_digit().
>
>2) Checks if the 'l' variable is >= to LONG_MAX (2,147,483,647)
>
>3) Multiplies the 'l' variable by 10. eg. l = 123; l *= 10 = 1230;
>
>4) Adds current character from 'c' variable to 'l' variable. ('c' is minus by '0' because 'c' is a char it needs to be converted to a int for 'l' to accept it)

After awhile of going around in circles in research and looking on the plaza forum, I started to get a sense of where to go. Because 'l' is a 8 byte variable putting 8 byte data into a 4 byte variable, has to have problems surrounding that. I came across information like two's complement and integer overflows. Now while I did know of these problems and had a feeling that this was going to be a problem from the start. I was still struggling because I didn't understand why it wouldn't cause such problems when I try to exploit them in the challenge.
