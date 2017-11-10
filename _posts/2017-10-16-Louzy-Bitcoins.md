---
layout: post
title: Lousy Bitcoins
categories: [Bitcoin, Python, Cryptography]
---

#### Cracking bitcoins private keys generated from weak passphrases.

I recently started digging deeper into the whole bitcoin/blockchain world. While reading the very good book [Mastering Bitcoins](http://amzn.to/2hp4uD0), I learnt the specifics of how bitcoin addresses and keys work. The generation process is made up of solid cryptographic building blocks, but if not carried out "right", there are possibilities for people to take over your bitcoin wallet.

<!--more-->

## The Theory

Everything starts with your private key, the one needed to spend your bitcoins. The private key is essentially a large number . A number that nobody could easily guess, ideally generated from a good source of randomness. Here's a sample of a private key in hex:

```
1E99423A4ED27608A15A2616A2B0E9E52CED330AC530EDCC32C8FFC6A526AEDD
```

The public key is derived from the private key by applying the following one-way functions, in order: *Eliptic Curve Multiplication and SHA256* (I know, quite big words). The bottom line is that by using those functions is computationally infeasible to retrieve the private key solely from the public key (that's why they are called *one-way*).

Getting the final bitcoin address also involves performing few cryptographic one-way functions over the public key, but this time the goal is mostly to arrive at a smaller and "nicer" format, something like:

```
14cHX5W3s58dCF7rsY9mzQBdK6HDAT8FDn
```

So the bitcoin address comes from a series of known transformations of the original private key. But you can safely make it public, without compromising the security of your wallet.

## The Problem

Don't get me wrong, the above protocol is very secure, but its entire security lies on initially generating a strong random number (the private key). Since it's a very large number, hard to remember, many people rely on so called brain wallets, which essentially takes a passphrase and turn them into your private key, simply as follow:

```
private key = sha256(passphrase)
```

In the early days of bitcoins, some brain wallet services used the method above, and if you used insecure passphrases, it would be easy for attackers to control your wallet. That's the password reuse problem all over again.

## Some Proof

It turns out that people actually used the method above to generate private keys! Here are some examples:

- ‘sausage’: [https://bitref.com/1LdgTMX2MEqdfT3VcDpX4GyD1mqCP8LkYe](https://bitref.com/1LdgTMX2MEqdfT3VcDpX4GyD1mqCP8LkYe)
- ‘test’: [https://bitref.com/1LxXC4tTyubWLAF9Z23FcMUNKAJfR5HbDK](https://bitref.com/1LxXC4tTyubWLAF9Z23FcMUNKAJfR5HbDK)
- ‘lol’: [https://bitref.com/1LuKiaNUT936SYHvErofFEqX1LAN6Z62Vg](ttps://bitref.com/1LuKiaNUT936SYHvErofFEqX1LAN6Z62Vg)

As you can see, all of them had, at some point, some balance available. But then, not long after that, it was gone.

## Try Yourself

So if one would had a list of passwords/passphrases people commonly use, then they could check them against the blockchain to see if there's any transaction history (or any balance if you like!).

[I wrote a python to demonstrate that.](https://github.com/joarleymoraes/lousy_bitcoins)

**Disclaimer**: I hold no liability for what you do with that code. I built this for learning purposes and to try to get people to be more careful with their private keys. Also no bitcoin address has been compromised while writing this.






