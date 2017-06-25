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

Example usage of SymPy:
```python
>>> from sympy import *
>>> x = symbols('x')
>>> e = x + 1
>>> e
x + 1
>>> e += x
>>> e *= x
>>> e
x*(2*x + 1)
>>> a = symbols('a')
>>> e += a*x
>>> e.subs({x: 2})
2*a + 10
```

Here I've defined a symbol `x` using the SymPy function `symbols()`. Then I created a expression `e` by adding `1` to `x`. Note that this doesn't trigger an actual addition, as `x` is a symbol with no specific value. Calling `e` simply prints the underlying expression. Expressions can be manipulated using standard arithmetic operators like `+`, `*` etc. To substitute a value for a symbol in an expression, the `subs` method can be used. The library also contains functions to manipulate polynomials. (polynomial division, extracting coefficients etc..)
```python
>>> p = 2*x**2 + x + 1
>>> g = x + 1
>>> div(p, g, x)
(2*x - 1, 2)
>>> Poly(p, x).coeffs()
[2, 1, 1]
```
In the above example, I've defined two polynomial expressions `p` and `g`. Then I used the `div` function to perform [poylnomial divison](https://en.wikipedia.org/wiki/Polynomial_long_division). The result is a tuple of (quotient, remainder).
The last statement extracts the coefficients of polynomial `p` and returns them as a list.

## Background

 Before describing my solution, I'll give a little bit of background about CRC codes for those who don't know / don't remember. CRC codes are based on division of polynomials which coefficients in the finite field GF(2).

### GF(2)
[GF(2)](https://en.wikipedia.org/wiki/GF(2)) is a [field](https://en.wikipedia.org/wiki/Field_(mathematics)) which contains only two elements, 0 and 1. Addition and multiplication are done *modulo 2*. For example, `0+1 = 1`, `1+1 = 0`, `0*1 = 0`, `1*1 = 1`. Addition is nothing but the [Boolean XOR](https://en.wikipedia.org/wiki/XOR_gate) operator.   

### Algebra in GF(2)
Since GF(2) is a field, we can do algebra in the same way as we do for rational and real numbers, except everything has to be done *modulo 2*. "Variables" in GF(2) are just Boolean variables (taking values of only 0/1). Let `a` & `b` be two variables in GF(2). If `e = a + b` & `f = a + 1`, then `e + f = b + 1`. Unfortunately, I could not find a built-in mechanism to do algebra in GF(2) using SymPy, so I implemented the following simple function:
```python
def modtwo(e, syms):
    coeffs = e.as_coefficients_dict()
    res = 0
    for sym in syms:
        res += (coeffs[sym]%2)*sym
    res += coeffs[Integer(1)]%2
    return res
```
It takes an expression and a list of symbols as input and returns a new expression with all the coefficients of the symbols in the input expression taken modulo 2. Hence, we can do all the algebra we need normally using SymPy and finally apply the `modtwo` fuction on the result to get the result we would have gotten if we had done the algebra in GF(2). (This is inefficient but in this situation we don't really have to care). For example to add the expressions `e` and `f` in the previous paragraph we can simply do:
```
>>> e = a + b
>>> f = a + 1
>>> modtwo(e+f, [a,b])
b + 1
```

### CRC

## Formulating the problem

## Doing the Algebra

## Solving the constraints

## Summary

## References
