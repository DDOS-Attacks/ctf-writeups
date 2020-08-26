# Challenge
Chunk Norris is black belt in fast random number generation.

We are given 2 files, a [challenge.py](challenge.py) and [output.txt](output.txt) file. It looks like the flag was encrypted with textbook RSA, and that the 2 primes were the modulus (both 1024-bit) were generated in an unsafe way.

The goal is to recover the primes and decrypt the ciphertext to get the flag.

## Weak Prime Generation

Let's a take look at the prime generatation and see why it's insecure.
```python
#!/usr/bin/python3 -u

import random
from Crypto.Util.number import *
import gmpy2

a = 0xe64a5f84e2762be5
chunk_size = 64

def gen_prime(bits):
  s = random.getrandbits(chunk_size)

  while True:
    s |= 0xc000000000000001
    p = 0
    for _ in range(bits // chunk_size):
      p = (p << chunk_size) + s
      s = a * s % 2**chunk_size
    if gmpy2.is_prime(p):
      return p

n = gen_prime(1024) * gen_prime(1024)
e = 65537
flag = open("flag.txt", "rb").read()
print('n =', hex(n))
print('e =', hex(e))
print('c =', hex(pow(bytes_to_long(flag), e, n)))
```

If we look closely at the `gen_prime` function, we can see a couple of things.

Every prime generated will have 3 bits always set, bit 1023, 1022, and 1. This means that every prime will be in the form:
```
p = 2^1023 + 2^1022 + 1 + ...
```
`chunk_size` is set to 64. With the way the prime is generated, we're taking 1024 bits, spliting it into 64-bit chunks (hence the name) and putting a 64-bit number in there. What this means is that are primes actually look like this:
```
p = (c1 * 2^960) + (c2 * 2^896) + (c3 * 2^832) + ... + c16
```
Where `c1, c2, ..., c16` are each individiual 64-bit chunk.

Essentially it's 16 different 64-bit numbers all merged together into 1 1024-bit number. The good news is, with the first and last chunk we can easily recover the s values and then regenerate the primes that were used. Let's take a look at the first chunk and how it's generated.

```python
s = random.getrandbits(64)
p = 0
p = (p << 64) + s = s
s = (a * s) % 2**64
```
So the first chunk is basically `s`. Every loop, `s` is updated by multiplying it with `a` and taking the value `mod 2**64`. Then this value is used in the next chunk. What this tells us is that the chunks have the form. 
```
c_k = (a^(k-1)*s) mod 2**64
c_1 = s
c_2 = (a*s) mod 2**64
c_3 = (a * ((a*s) mod 2**64)) mod 2**64 = (a^2*s) mod 2**64
...
c_16 = (a^15*s) mod 2**64
```
We only need to take the `mod 2**64` once, as doing it multiple times will arrive at the same result. With this new knowledge, we can easily recover s. But there's an extra step; this is the form for one of the primes, but `n` is the product of 2 of these primes! So we have to apply a few math tricks to figure out `s`.
```
N = p*q
N = (