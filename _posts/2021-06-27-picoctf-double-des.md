---
layout: post
title:  "picoCTF 2021- double-des"
date:   2021-06-27 16:00:00 +0200
categories: ctf crypto writeups
excerpt_separator: <!--more-->
mathjax: true
---

This March I've partecipated to [picoCTF](https://picoctf.org/) with a couple of friends as my first "official" CTF. It was a lot of fun and I encountered some really interesting problems so I've decided to spend some time making a writeup to improve my English and to return some life into this blog.<!--more--> Let's start with a simple yet interesting problem.

## picoCTF 2021 - double-des

# Description

    I wanted an encryption service that's more secure than regular DES, 
    but not as slow as 3DES... The flag is not in standard format. 

We also have an address and a python script. I will not report the entire script, the important things in it are:

- It encrypts the flag with a random key then encrypt the result with a second, different random key. These keys are 6-digits numbers.
- The script print the encrypted flag.
- The script waits for our input. 
- After the user submits an input the script encrypt the input and it prints the ciphertext

So, we have the ability to do a chosen plaintext attack but how will we do it? A single key is simple enough to bruteforce it (\\( 10^6 \\)) however when we have two keys we have \\(10^{12}\\) possible solutions.

If you've ever followed a cryptography class you already know where is the issue in the algorithm the server is using: while it's true that the possible keys are \\(10^{12}\\) we don't need to try them all: we can do a meet-in-the-middle attack.

# Meet-in-the-middle

The idea behind this attack is this:

- We use our oracle to encrypt \\(P\\) (chosen plaintext), obtaining \\(C\\)
- We try to encrypt \\(P\\) using random keys and we keep the list of the generated \\(C_{key1}\\)
- We then try to decrypt \\(C\\) using random keys. For each \\(P_{key2}\\) we check the list created in the previous step to see if \\(P_{key2}\\) is present in the list. If we can found it, it means that we have \\(C_{key1} = P_{key2}\\) and we have found our keys.

The reason why this works it's pretty simple: since the input of the second DES function is the output of the first one, we can try "both sides" to see which keys generate the correct middle value. For doing so, we must try at most 2 times the key size, reducing the amount of work needed. This is also the reason why we teach cryptography students that you can't simple continue to encyrpt the ciphertext (when using a block cipher) to increase its confidentiality!

Back to our problem, using a meet-in-the-middle attack we can reduce the complexity of our attack to at most \\(10^7\\) key checks which is feasable.

# My solution

I've implemented a meet-in-the-middle attack and since the keyset is so small (only \\(10^6\\)) I've decided to use a brute force approach. More precisely:

- I've choosed a plaintext ("11111111") for semplicity and I've submitted to the service, obtaining its ciphertext

```python
chosen_plaintext = pad(binascii.unhexlify("11111111").decode())
ciphertext = b"\x0c\xa5\xb4\x97\x6d\x66\xeb\x57"
```

- I've tried all \\(10^6\\) keys to encrypt this plaintext using DES. Each result is the key to a dictonary containing, as value, the DES key used to generate the ciphertext

```python

partial_ciphertexts = {}

for first_key in range(0,1000000):
    des = DES.new(pad(str(first_key).zfill(6)), DES.MODE_ECB)
    c = des.encrypt(chosen_plaintext)
    partial_ciphertexts[c] = first_key
```

3) I've tried all \\(10^6\\) keys to decrypt the ciphertext. For each result I've checked if there is a value in the dictonary for that key. If I can find it then we have our two keys: the one in the dictonary for the first pass and the second one currently in use for the second pass.

```python
for second_key in range(0, 1000000):
    des = DES.new(pad(str(second_key).zfill(6)), DES.MODE_ECB).decrypt(output)
    c = des.decrypt(ciphertext)
    try:
        first_key = partial_ciphertexts[c]
        print("found keys!")
        print(str(first_key).zfill(6) + ' , ' + str(second_key).zfill(6))
        print("Decrypting flag...")
        decrypt_twodes(flag, first_key, second_key)
    except KeyError:
        continue
```

4) With the two keys, we can now easily decrypt the flag.

```python
def decrypt_twodes(flag, first_key, second_key):
    de1 = DES.new(pad(first_key), DES.MODE_ECB)
    c = cipher1.decrypt(flag)
    des2 = DES.new(pad(second_key), DES.MODE_ECB)
    plaintext_flag = cipher2.decrypt(enc_msg)
    print(plaintext_flag.decode())
    print(binascii.hexlify(plaintext_flag))
```

With that we have our flag and we have completed this challange.
