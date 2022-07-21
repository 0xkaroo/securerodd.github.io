---
layout: post
title: 'Paradigm CTF 2021: babycrypto'
category: Paradigm CTF 2021
excerpt_separator:  <!--more-->
---

### Challenge:
This write-up is for the challenge titled 'babycrypto'.


This challenge is located in the babycrypto/public/ directory. The source code can be found below:

<details>
<summary> chal.py:</summary>
<br>
<div markdown="1">
```
from random import SystemRandom
from ecdsa import ecdsa
import sha3
import binascii
from typing import Tuple
import uuid
import os


def gen_keypair() -> Tuple[ecdsa.Private_key, ecdsa.Public_key]:
    """
    generate a new ecdsa keypair
    """
    g = ecdsa.generator_secp256k1
    d = SystemRandom().randrange(1, g.order())
    pub = ecdsa.Public_key(g, g * d)
    priv = ecdsa.Private_key(pub, d)
    return priv, pub


def gen_session_secret() -> int:
    """
    generate a random 32 byte session secret
    """
    with open("/dev/urandom", "rb") as rnd:
        seed1 = int(binascii.hexlify(rnd.read(32)), 16)
        seed2 = int(binascii.hexlify(rnd.read(32)), 16)
    return seed1 ^ seed2


def hash_message(msg: str) -> int:
    """
    hash the message using keccak256, truncate if necessary
    """
    k = sha3.keccak_256()
    k.update(msg.encode("utf8"))
    d = k.digest()
    n = int(binascii.hexlify(d), 16)
    olen = ecdsa.generator_secp256k1.order().bit_length() or 1
    dlen = len(d)
    n >>= max(0, dlen - olen)
    return n


if __name__ == "__main__":
    flag = os.getenv("FLAG", "PCTF{placeholder}")

    priv, pub = gen_keypair()
    session_secret = gen_session_secret()

    for _ in range(4):
        message = input("message? ")
        hashed = hash_message(message)
        sig = priv.sign(hashed, session_secret)
        print(f"r=0x{sig.r:032x}")
        print(f"s=0x{sig.s:032x}")

    test = hash_message(uuid.uuid4().hex)
    print(f"test=0x{test:032x}")

    r = int(input("r? "), 16)
    s = int(input("s? "), 16)

    if not pub.verifies(test, ecdsa.Signature(r, s)):
        print("better luck next time")
        exit(1)

    print(flag)
```
</div>
</details>


The hints and solutions for this level can be found below:

<details>
<summary> Hint 1:</summary>
<br>
<div markdown="1">
Read up on [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) 
</div>
</details>

<details>
<summary> Hint 2:</summary>
<br>
<div markdown="1">
(https://medium.com/asecuritysite-when-bob-met-alice/not-playing-randomly-the-sony-ps3-and-bitcoin-crypto-hacks-c1fe92bea9bc)
</div>
</details>


<summary> Solution:</summary>
<br>
<div markdown="1">
ECDSA algorithms require that the k value not only be sufficiently random, but also that it be different for each signature created. In this challenge, a session secret was generated and then it was used to sign four different messages. 

This allowed us to derive k and then eventually to derive the private key. Using the private key and k, we can in turn recover both r and s and solve the challenge.
</div>
</details>

<details>
<summary> Solution.py:</summary>
<br>
<div markdown="1">
```
from ecdsa import ecdsa
import chal as c
from mp import * 

# Starting the process
p = process('python3', 'chal.py')

# Sending in test1 as our message and collecting our first r and s values
p >> 'message? ' << 'test1\n' >> 'r='
r1 = int(p.recvline(), 16)
p >> 's='
s1 = int(p.recvline(), 16)
m1 = "test1"

# Print out our first pair of r and s
print(f"r1: 0x{r1:x}")
print(f"s1: 0x{s1:x}")

# Sending in test2 as our message and collecting our second r and s values
p >> 'message? ' << 'test2\n' >> 'r='
r2 = int(p.recvline(), 16)
p >> 's='
s2 = int(p.recvline(), 16)
m2 = "test2"

# Print out our second pair of r and s
print(f"r2: 0x{r2:x}")
print(f"s2: 0x{s2:x}")

# Creating our z values from the messages
z1 = c.hash_message(m1)
z2 = c.hash_message(m2)

# Ensuring we have the correct order for our calculations
g = ecdsa.generator_secp256k1
n = g.order()

# Deriving k and the private key (da) due to k reuse
k = ((z1 - z2) % n ) * (ecdsa.numbertheory.inverse_mod(s1 - s2, n)) % n
da = ((((s1 * k) % n) -z1) * ecdsa.numbertheory.inverse_mod(r1, n)) % n

# Sending two more messages and then gathering our final hash
p << 'test3\ntest4\n' >> "test="
z3 = int(p.recvline(), 16)
print(f"test: 0x{z3:032x}")

# Calculations to recover r based on k (note, r is already known so we verify this later)
new_k = k % n
p1 = new_k * g
r = p1.x() % n
assert r == r1 == r2

# Calculation to recover s based on r, our final hash and the private key
s = (ecdsa.numbertheory.inverse_mod(k, n)* (z3 + (da * r) % n)) % n

# Send the r and s values
p >> 'r? ' << hex(r) << '\n'
p >> 's? ' << hex(s) << '\n'

# Gather flag and print to terminal
flag = p.recvline().decode('utf8')
print(f"flag: {flag}")
```
</div>
</details>