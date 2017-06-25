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

 Before describing my solution, I'll give a little bit of background about CRC codes for those who don't know / don't remember. CRC codes are based on division of polynomials with coefficients in the finite field GF(2).

### GF(2)
[GF(2)](https://en.wikipedia.org/wiki/GF(2)) is a [field](https://en.wikipedia.org/wiki/Field_(mathematics)) which contains only two elements, 0 and 1. Addition and multiplication are done *modulo 2*. For example, `0+1 = (0+1) mod 2 = 1`, `1+1 = 0`, `0*1 = (0*1) mod 2 = 0`, `1*1 = 1`. Addition is nothing but the [Boolean XOR](https://en.wikipedia.org/wiki/XOR_gate) operator.   

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
```python
>>> e = a + b
>>> f = a + 1
>>> modtwo(e+f, [a,b])
b + 1
```

### CRC
[Cyclic Redundancy Check (CRC)](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) is a [checksum](https://en.wikipedia.org/wiki/Checksum) that is commonly used for error detection and correction in numerous applications. The input is a binary message of arbitrary size and the output is a fixed size checksum.

Consider a polynomial with coefficients in GF(2) (I'm going to call such a polynomial as *GF(2) Polynomial*). Since each of the terms' coefficients is either 0/1, we can represent it as a bit string of length degree + 1. For example the polynomial `x**3 + x + 1` would be represented as `1011`. Similarly every bit string can be represented as a polynomial (with coefficients in GF(2)) of degree length - 1. Hence, there is a bijective mapping between bit strings and GF(2) polynomials.

To compute the CRC checksum of a bit string it is converted to it's corresponding GF(2) polynomial which is then divided by a fixed *generator polynomial*. The remainder polynomial obtained after division is converted back to a bit string which is the checksum. Different variants of CRC may use different generator polynomials. Also, before division the input bitstring is left shifted by an amount equal to the degree of the generator polynomial. Since the coefficients are in GF(2), while performing the polynomial division, all arithmetic on the coefficients must be done modulo 2 (which is equivalent to doing it normally and taking all the coefficients modulo 2 and the end).

For example, assume the generator polynomial is `G(x) = x**3 + x + 1`. Then to compute the CRC checksum of bitsring `11010011101` (polynomial form: `x**10 + x**9 + x**7 + x**4 + x**3 + x**2 + 1`):
```python
>>> g = x**3 + x + 1 # Generator polynomial 
>>> p = x**10 + x**9 + x**7 + x**4 + x**3 + x**2 + 1 # Input polynomial
>>> p *= x**3 # Left shift
>>> remainder = div(p, g, x)[1] # Remainder after division
>>> coeffs = Poly(remainder, x).all_coeffs()
>>> [c % 2 for c in coeffs] # Coefficients of remainder modulo 2
[0 0 1]
```
Hence the CRC checksum is `001`. Note that the number of bits of the checksum will always be equal to the degree of the generator polynomial.

## Formulating the problem
Recap of the problem: Find a string of '0's and '1's of length 82 whose CRC checksum when represented in binary is the string itself. The CRC variant we need to deal with is the `crc_82_darc` function from the pwnlib library. It is easy to find the details of this function looking at the [docs]() and [source]() of pwnlib:
* The generator polynomial (in bitstring form) is `0x308c0111011401440411 | (1<<82)` (degree 82)
* Before division, the *bits of every byte* in the input string are reversed
* The bits of the resulting checksum are reversed before it is returned

Let the string we are looking for be *b<sub>1</sub>b<sub>2</sub>b<sub>3</sub>..b<sub>82</sub>* where each of the *b<sub>i</sub>*s is a boolean variable representing whether the i<sup>th</sup> character in the string is a '0' or '1'. Since the string has 82 characters the number of bits is 8x82. The binary representation of '0' is `00110000` and '1' is `00110001`. Since `crc_82_darc` first reverses the bits in every byte, their representations effectively becomes `00001100` & `10001100` respectively. Hence in general the i<sup>th</sup> byte of the input bitstring is *b<sub>i</sub>0001100*. Lets construct the polynomial corresponding to the input bitstring *m(x)*:
```python
x = symbols('x')
b = ['']
for i in xrange(1, 83):
    b.append(symbols('b' + str(i)))

m = 0 # Input polynomial
zero = [0,0,0,1,1,0,0]
k = 0
for i in xrange(82, 0, -1):
    for c in zero[::-1]:
        m += c*(x**k)
        k += 1
    m += b[i]*(x**k)
    k += 1
```

Next, let's construct the generator polynomial *g(x)* from it's bitstring form:
```python
polynom = 0x308c0111011401440411 | (1<<82)
polynom = bin(polynom)[2:]
polynom = [int(bit) for bit in polynom]
g = 0 # Generator polynomial
k = 0
for bit in polynom[::-1]:
    g += bit*(x**k)
    k += 1
```

Since `crc_82_darc` reverses the bits of the result before returning, we need the remainder after division to be *b<sub>82</sub>b<sub>81</sub>b<sub>80</sub>....b<sub>1</sub>*. To construct the polynomial form of that:
```python
r = 0 # Remainder polynomial
for i in xrange(82):
    r += b[i+1]*(x**i)
```

Now that all the required polynomials (*m(x)*, *g(x)*, *r(x)*) have been defined, all we have to do is find *b<sub>i</sub>*s such that `m(x)*x**82 = q(x)*g(x) + r(x)` for some polynomial q(x). 

## Doing the Algebra
The above equation can be re-written as `m(x)*x**82 - r(x) = q(x)*g(x)`: The remainder when `m(x)*x**82 - r(x)` is divided by `g(x)` is `0`. We can now let SymPy do the tedious polynomial division for us:
```python
m *= x**82
remainder = div(m - r, g, x)[1]
```
This may take 1-2 minutes to execute, because the coefficients of the *b<sub>i</sub>*s end up becoming large. (Again this is not an efficient way of doing polynomial division in GF(2), but we don't need to worry about that here since it's a one time computation). Now we extract the coefficients of the `x` terms in the polynomial and apply `modtwo` on them to get the actual coefficient expressions:
```python
remp = Poly(remainder, x)
c = remp.coeffs()

final = []
for coeff in c:
    final.append(modtwo(coeff, b[1:]))
```
`final` now contains a list of 82 expressions for each of the coefficients of the remainder polynomial. The coefficients of each *b<sub>i</sub>* in the expressions are either 0/1. Since we want the remainder to be `0`, each of these expressions must equal `0`. Hence, we have *82* equations in *82* boolean variables. Solving these will lead us to our answer. 

## Solving the constraints
The left hand side of each equation is a sum of a set of boolean variables and the right hand side is either 0/1. Since, addition in GF(2) is nothing but the boolean XOR operator, these are essentially *XOR constraints*. A cyrpto-optimized SAT solver should easily be able to solve for these constraints. Since GF(2) is a field, solver's don't need to brute-force over all possible variable assignments. Instead they can use [Gaussian Elimination](https://en.wikipedia.org/wiki/Gaussian_elimination) to solve the system of linear equations in GF(2).

I used [CryptoMiniSAT](https://github.com/msoos/cryptominisat) for this purpose because is support gaussian elimination while dealing with XOR constraints and has a very simple python interface. (It needs to be compiled with the right flags to enable gaussian elimination). Using the interface we can easily construct XOR constraints from the previously obtained expressions:
```python
from pycryptosat import Solver
s = Solver(verbose = 1)
for eqn in final:
    lhs = []
    for i in xrange(1, 83):
        if eqn.coeff('b' + str(i)) == 1:
            lhs.append(i)

    rhs = (eqn.as_coefficients_dict()[Integer(1)] == 1)
    s.add_xor_clause(lhs, rhs)
```

Upon firing the SAT solver, it spits out the solution almost instantaneously:
```python
sat, sol = s.solve()
print ''.join([str(int(x)) for x in sol[1:]])
```

The solution is:
`1010010010111000110111101011101001101011011000010000100001011100101001001100000000`
Submitting it to the prompt we get our flag: `CTF{i-hope-you-like-linear-algebra}`
Absolutely not! The authors are probably refering to the gaussian elimination which the SAT solver was using under the hood. Luckily for me SymPy & CryptoMiniSAT did all the heavy lifting, and I didn't have to explicitly do any linear algebra :) 

## Summary
