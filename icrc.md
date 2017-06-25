# Introspective CRC

This is a write-up of my solution to an interesting crypto challenge that appeared in Google CTF 2017 Quals. The challenge asks us to netcat to a particular location and upon doing so a prompt asking for data comes up:
```
$ nc selfhash.ctfcompetition.com 1337
Give me some data:   
```
Entering a random string reveals the first hint (it's a python expression):
```
$ nc selfhash.ctfcompetition.com 1337
Give me some data: wobuffet

Check failed.
Expected:
    len(data) == 82
Was:
    8
```
The input string must be of length 82 characters. After inputting 82 'a's:
```
$ python -c "print 'a'*82" | nc selfhash.ctfcompetition.com 1337
Give me some data: 
Check failed.
Expected: 
    set(data) <= set("01")
Was:
    set(['a'])
```
This means the string must contain only '0's and '1's. Satisfying that, we get the last and final hint:
```
python -c "print '0'*82" | nc selfhash.ctfcompetition.com 1337
Give me some data: 
Check failed.
Expected: 
    crc_82_darc(data) == int(data, 2)
Was:
    1021102219365466010738322L
    0
```
A quick google search for `crc_82_darc` tells us that it's a function from the `pwnlib` python library. It's a variant of the **CRC checksum** which is commonly used for error detection. (Think MD5 but much much much much simpler). The function takes a string and outputs a number (82 bit in this case). According to the hints we need to find a string of '0's and '1's of length 82 whose CRC checksum when represented in binary is the string itself. This challenge was inspired from a "self hashing" [GIF image](https://shells.aachen.ccc.de/~spq/md5.gif) which was popular on twitter, reddit and hacker news a few months ago. The MD5 hash of the image file is the same as the text displayed on the image. Analogously, here we need to find a string (as opposed to image) which hashes to itself (CRC as opposed to MD5).

Bruteforcing over all possible strings is clearly infeasible (there are 2^82 possibilities). Trivial strings like all '0's, all '1's, all '0's except one '1' etc... dont work either. Hence, I decided to take a more principled approach and mathematically formulate the problem. It turns out that doing so requires a fair amount of algebra. To deal with this I made use of SymPy, a really cool python library that let's you programmatically do all the algebra you need without the need of pen and paper.

## SymPy
[SymPy](http://www.sympy.org/en/index.html) is python library for symbolic computation. It basically lets you do algebra at a higher level of abstraction. You can define symbols and construct expressions using them. To perform operations on expressions, you just have to tell SymPy what operation you want to do and it does all the heavy-lifting of actually performing the algebra, returning only the final expression to you. While working with large expressions doing the algebra on paper can become tedious and SymPy can really help in such situations.

[TODO: Examples of SymPy usage]

## Background

### GF(2)

### Algebra in GF(2)

### CRC

## Formulating the problem

## Algebra

## SAT Solving

## References
