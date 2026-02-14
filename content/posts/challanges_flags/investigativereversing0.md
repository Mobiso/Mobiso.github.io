+++
title = 'Investigative Reversing 0'
date = 2026-02-10T13:29:36+01:00
draft = false
description = "My write up for PicoCTF flag that was rated as Hard"
+++
>Author: Danny Tunitis
>
> ## Description
>We have recovered a binary and an image. See what you can make of it. There should be a flag somewhere.

# Solution
First I loaded up the binary in `Ghidra`. 
![Mystery binary loaded up in Ghidra](/pico/investigativereversing0/ghidra_raw.png)
Ghidra gave me some nice C code to work with and I began renaming variables that were obvious, such as arguments to the standard functions and what looked like for loops. This made it much easier to read.
![Named Variables In mystery binary](/pico/investigativereversing0/ghidra_named_variables.png)
It requires flag file, read 26 bytes from it and loads it into a buffer. It then appends 4 bytes of the flag to the mystery image and 2 mystery bytes. It then appends 9 bytes but adds 5 to each character followed by adding a mystery byte again where it subtracts 3. It then proceeds to add the rest of the flag bytes. 

I used the `tail -c 26 original.png | hexdump -c` command on the mystery image and behold part of the flag revealed itself:
![Encoded flag](/pico/investigativereversing0/encodedflag.png)

But what about the mystery bytes added? Lets see what happens if I give it a flag consisting of 26 1:s:
![Flag consisting of 1:s](/pico/investigativereversing0/mysterbyterevealed.png)
It appears that the mystey bytes 5 and 6 are just bytes from the flag. The same with the mystery byte where 3 was subtracted. If we look at the ascii table the value for `.` is 46, 46 + 3 is 49 and the value for `1` is 49. Since we know which bytes 5 was added to and what byte 3 was subtracted from, it is just a matter of looking up the correct value in the ascii table and subtract 5 and add 3 to the correct characters. This gives you the flag: 
>picoCTF{f0und_1t_dbf6ab4d}