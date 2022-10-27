# Cult Meeting
Hack the Boo 2022

## Description

```
After months of research, you're ready to attempt to infiltrate the meeting of a shadowy cult. Unfortunately, it looks like they've changed their password!
```

## Challenge Explanation

For this challenge, we were given a Netcat connection as well as a compiled binary. The goal was to find the password and use it in the Netcat session.

## The Vulnerability

With reversing challenges, it's always worth it to run `strings` on the binaries. This tool will print out anything from a file that could be interpreted as a string. Any sequence of bytes that is at least 4 characters long and corresponds to valid ASCII will be printed. On the huge majority of reversing challenges beyond the absolute basics, it doesn't yield anything, but it's also zero-effort so there's no reason not to try it.

Running `strings` on this file yielded the following (along with a bunch of other junk, of course):

```
[3mYou knock on the door and a panel slides back
[3m A hooded figure looks out at you
"What is the password for this week's meeting?" 
sup3r_s3cr3t_p455w0rd_f0r_u!
[3mThe panel slides closed and the lock clicks
|      | "Welcome inside..." 
/bin/sh
   \/
 \| "That's not our password - call the guards!"
```

And there's our password. Now we can connect to the remote Netcat session and use the password `sup3r_s3cr3t_p455w0rd_f0r_u!`. Once you've given the password, the program prints the flag.

Tragically, this one was so quick that I didn't even bother to write down the flag when I got it... I just submitted it and moved on. Whoops :(