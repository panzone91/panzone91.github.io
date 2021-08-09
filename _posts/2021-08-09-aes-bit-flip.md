---
layout: post
title:  "imaginaryCTF 2021 - flop"
date:   2021-08-09 21:30:00 +0200
categories: ctf crypto writeups
excerpt_separator: <!--more-->
mathjax: true
---

A couple of weeks ago while playing a CTF I've learned a new kind of vulnerability to AES, which means it's time for a new writeup.<!--more-->

## imaginaryCTF 2021 - flop

# Description

    Yesterday, Roo bought some new flip flops. 
    Let's see how good at flopping you are.

We also have the source code of the server:

```python
...
key = os.urandom(16)
iv = os.urandom(16)
flag = open("flag.txt").read().strip()


for _ in range(3):
	print("Send me a string that when decrypted contains 'gimmeflag'.")
	print("1. Encrypt")
	print("2. Check")
	choice = input("> ")
	if choice == "1":
		cipher = AES.new(key, AES.MODE_CBC, iv)
		pt = binascii.unhexlify(input("Enter your plaintext (in hex): "))
		if b"gimmeflag" in pt:
			print("I'm not making it *that* easy for you :kekw:")
		else:
			print(binascii.hexlify(cipher.encrypt(pad(pt, 16))).decode())
	else:
		cipher = AES.new(key, AES.MODE_CBC, iv)
		ct = binascii.unhexlify(input("Enter ciphertext (in hex): "))
		assert len(ct) % 16 == 0
		if b"gimmeflag" in cipher.decrypt(ct):
			print(flag)
		else:
			print("Bad")

print("Out of operations!")
```
Ok, to obtain the flag we must make the server decrypt a string containing *gimmeflag*. We can use the server to encrypt a string but our plaintext can't contain *gimmeflag* so we can't simply encrypt a string and then give it back to the server. We can execute only 3 operations before the server changes the key. The server uses AES in CBC mode using a random IV for the encrypt-decrypt operations, so let's see how it works.

# AES - CBC

A mode of operation describes how to use a fixed-size algorithm like AES to encrypt a longer plaintext. CBC is one of the more commons mode of operation for this kind of jobs. 

<img src="/assets/blog/images/cbc.png"/>

The idea is that each block of the plaintext is XOR-ed with the previous ciphertext. The first block is XOR-ed with an Inizialization Vector (IV). More precisely, if we consider \\(C_{0}\\) as the IV we can describe each block as:

\\(C_{i} = E_{k}(P_{i}) \oplus C_{i-1}\\)

\\(C_{0} = IV\\)

and each decrypted blocks as:

\\(P_{i} = D_{k}(C_{i}) \oplus C_{i-1}\\)

\\(C_{0} = IV\\)

Just from this definition we can already see an issue with CBC: the structure of the message doesn't change between the plaintext and the ciphertex. The encryption of the second block of plaintext, for example, will always be the second block of the ciphertext. If we know the structure of the plaintext we can try to use this fact to our advantage.

# AES - CBC bit flipping

Let's assume an attacker wants to manipulate the *i* block of plaintext, for example with a XOR operation. Using the equations above:

\\(P_{i}^n = P_{i} \oplus x\\)

\\(P_{i}^n = D_{k}(C_{i}) \oplus C_{i-1} \oplus x\\)

but

\\(P_{i}^n = D_{k}(C_{i}) \oplus (C_{i-1}^n)\\)

so

\\(C_{i-1}^n = C_{i-1} \oplus x\\)

It means that if we apply a XOR operation to the \\(C_{i-1}\\) block the same operation is then applied the *i* block of the plaintext. If we know the structure of the plaintext we can then modify the ciphertext to change the decrypted plaintext as we want.

# gimmeflag

At this point is easy what we need to do to solve this challange: 

- we create a plaintext bigger than a single block (so bigger than 16 characters). This plaintext also needs to have the string we want to change. I will use as a string *ccccccccccccccccgimmeflaf*, 16 *c*-s and the string *gimmeflaf*. 
- we encrypt the plaintext and save the ciphertext
- we change the ninth byte of the ciphertext with a \\(\oplus 0x01\\) (which will flip the last bit of the ninth byte). This will change the last bit of the ninth byte of the second block of the plaintext, converting our *f* in a *g* 

```python
...
sh = remote('chal.imaginaryctf.org', 42011)

plaintext = b'c' * 16 + b'gimmeflaf'

# Wait until we can send our input
waitingForInput()

# Send plaintext
sh.sendline(b'1')
sh.sendline(binascii.hexlify(plaintext))

ciphertext = binascii.unhexlify(sh.recvline());
# We want to change the 9th byte with a XOR 1 (f -> g)
xor_key = (b'\0'* 8) + (b'\x01') + (b'\0'* 23)

changed_ciphertext = xor(ciphertext, xor_key)

waitingForInput()

sh.sendline(b'2')
sh.sendline(binascii.hexlify(changed_ciphertext))
flag = getFlagFromStin(sh.recvline())
print(flag)
```

This procedure let us to decrypt a string with *gimmeflag* and we solved the challange.
