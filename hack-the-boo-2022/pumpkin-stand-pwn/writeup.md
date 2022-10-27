# Wrong Spooky Season
Hack the Boo CTF 2022

## Description

```
This time of the year, we host our big festival and the one who craves the pumpkin faster and make it as scary as possible, gets an amazing prize! Be fast and try to crave this hard pumpkin!
```

## Challenge Explanation

Time to crave that pumpkin. This challenge gave us a netcat connection to a simple command-line tool to buy pumpkin carving items:

```
                                          ##&
                                        (#&&
                                       ##&&
                                 ,*.  #%%&  .*,
                      .&@@@@#@@@&@@@@@@@@@@@@&@@&@#@@@@@@(
                    /@@@@&@&@@@@@@@@@&&&&&&&@@@@@@@@@@@&@@@@,
                   @@@@@@@@@@@@@&@&&&&&&&&&&&&&@&@@@@@@&@@@@@@
                 #&@@@@@@@@@@@@@@&&&&&&&&&&&&&&&#@@@@@@@@@@@@@@,
                .@@@@@#@@@@@@@@#&&&&&&&&&&&&&&&&&#@@@@@@@@@@@@@&
                &@@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&&&@@@@@@@@@@@@@@@
                @@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&&&&@@@@@@@@@&@@@@@
                @@@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&&&@@@@@@@@@@@@@@@
                @@@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&&&@@@@@@@@@@@@@@@
                .@@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&&&@@@@@@@@@@@@@@
                 (@@@@@@@@@@@@@@&&&&&&&&&&&&&&&&&@@@@@@@@@@@@@@.
                   @@@@@@@@@@@@@@&&&&&&&&&&&&&&&@@@@@@@@@@@@@@
                    ,@@@@@@@@@@@@@&&&&&&&&&&&&&@@@@@@@@@@@@@
                       @@@@@@@@@@@@@&&&&&&&&&@@@@@@@@@@@@/

Current pumpcoins: [1337]

Items: 

1. Shovel  (1337 p.c.)
2. Laser   (9999 p.c.)

>>
```

We only have enough pumpcoins to buy the shovel, but when we do, we get the following message:

```
How many do you want?

>> 1

Current pumpcoins: [0]

Good luck carving that hard pumpkin with a shovel!
```

It seems that the goal of the challenge is to buy the laser.

## The Vulnerability

Any time I see a CTF challenge where the goal is to buy stuff with money you don't have, my mind jumps directly to an integer overflow. Computers store values in set amounts of space, and when they run out of space to hold a value, undefined behaviour can crop up, such as positive number going negative, or very small numbers suddenly becoming very large. If our 'wallet' of 1337 pumpcoins can be overflowed (or underflowed), maybe we can trick the system into thinking we have more cash than we actually do.

In C, the most common, naive way to store numerical values is with the data type `int`. This is a 32-bit signed value, which means that one bit stores the sign (positive or negative) and 31 bits store the number. This functionally means that a C `int` can only store values between -2,147,483,648 and 2,147,483,647.

For example, if you have the number 2,147,483,646 stored in a C int, the bits will look like this. Note that the top bit is a 0, which indicates a positive number:

```
0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0
```

Add one to this number, and you get 2,147,483,647:
```
0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
```

Add one more again, and you get this nice number...
```
1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0
```

which is equal to -2,147,483,648???

Yep. Adding one more to the int when it was already at its max value has caused us to accidentally carry into the top bit, which determines the positive or negative sign. Now the computer interprets this as an aboslutely huge negative number, instead of an absolutely huge negative number.

Note that this works in reverse too. If you had a huge negative number, and subtracted from it, you'd suddenly flip up to a huge positive number. So maybe if we can get our account balance to go waaaay below 0, we can trick the system into thinking we have a huge positive balance. This version is called an underflow.

## The Solution

At this point, I had no confirmation that the program was using signed 32-bit ints, or even that it was written in C. However, the only way to find exploits is to try stuff, so I started tossing stuff in. I decided to try sending 2,147,483,647, the maximum positive int. I figured that it probably wouldn't be exactly right, but if there was an overflow vuln, this number is big enough that it will *definitely* cause the overflow on any positive value. For that reason, it was a useful smoke test.

This is where things got surprising and weird. I had initially intended to overflow in the prompt where I was asked for the number of items. I was hoping that asking for two billion shovels would put my account balance way into the red.

However, I messed up, and sent the 2,147,483,647 value when prompted to select an item. This is what happened: 

```
Current pumpcoins: [1337]

Items: 

1. Shovel  (1337 p.c.)
2. Laser   (9999 p.c.)

>> 2147483647

How many do you want?

>> 1

Current pumpcoins: [-20741]


[-] Not enough pumpcoins for this!
```

Frankly, this is bizarre. I'm not exactly sure what has caused this to occur, but my best guess is that the system was set up to read from the item number buffer in a loop, causing me to buy multiple instances of the items, all in one go. Either way... my account balance has indeed gone way negative? So that's good?

This was definitely overflowing *something*, so I decided to try it again. Here's what I got: 

```
Current pumpcoins: [-20741]

Items: 

1. Shovel  (1337 p.c.)
2. Laser   (9999 p.c.)

>> 2147483647

How many do you want?

>> 1

Congratulations, here is the code to get your laser:

HTB{1nt3g3R_0v3rfl0w_101_0r_0v3R_9000!}
```

Huh. It looks like asking for item 2,147,483,647 again has not only underflowed my account balance, it also conveniently bought my laser! I don't think this was the intended solution, but I'll take it.

Success!