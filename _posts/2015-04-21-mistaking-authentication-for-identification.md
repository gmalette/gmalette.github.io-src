---
layout: post
title: Mistaking Authentication for Identification
date: 2015-04-21 20:00:00 +0400
categories:
- General
---

*Disclaimer: I am not a security expert, nor a systems expert. The following text is tinted by my understanding and experience, and probably contains mistakes or misunderstandings.*

Earlier this year, I accidentally uncovered a flaw in GitHub, Heroku and many other similar providers. The flaw essentially allows a denial of service to some users with little effort from the attacker. The situation has not been fixed, and it likely will not be. At the center of the problem is the fundamental mistake of thinking that authentication can be a substitute for identification.

**Identification** is a process through which one claims an identity. One person may have several identities. For example, GitHub knows me under the identity of _gmalette_. My gamer tag on Steam is _Wako_. Anyone can claim those identities; anyone can try to login on GitHub using _gmalette_ or on Steam using _Wako_. We need a way to verify the authenticity of that identity claim, a way to make sure the user really is who they pretend to be.

**Authentication** is the process through which one verifies a claimed identity. Typically, the person being authenticated will produce a document that assures they own the identity. Such documents can be a passport, a credit card’s CVV code, a password, etc.

On the internet, we generally use password authentication. That is, when you connect to a website, you provide your identity (username, email address, etc). The server will find the account matching that identity, and use a password for authentication.

What happens when you mistake authentication for identification? What if somehow, your identity was directly related to the authentication mechanism? You can imagine a website where you only have to enter a password to be logged in. The password will be hashed and salted for security, but the system will keep the first 3 characters to identify the user. When a user logs in using a password, the system will find the user using the first 3 characters, and then authenticate using the full password.

Imagine Alice choses the password "123456" (identified by "123"), Bob chooses the password "password" (identified by "pas"). Bob and Alice can both use the system without any problem. That is, until Alice changes her password to "password123456" (also identified by "pas"). Now when Bob enters "password", he will be wrongly identified as Alice, but will be unable to authenticate because he does not know the password. Bob will be locked out of his account even if it had previously worked. Such a system couldn’t exist, could it?

Most Git shells I’ve interacted with use a similar process of identification. More specifically, most Git shells identify the user while negotiating an SSH key, before the authentication happens. During the negotiation process, the client SSH will find all the keys it knows. For each key, it will send the fingerprint of the **public** key to the server. If the key is known to the server, it will send a challenge and SSH will use the **private** part of the key to respond to the challenge. The challenge may vary based on a few factors, but it always serves to prove that the user being authenticated owns the private key. One type of proof could be to sign a random piece of data. Signing is an operation that can only be performed using the private key, and easily verified using the public key. If a malicious user sends a known fingerprint but can’t respond to the challenge (because they don’t have the private key), they will not be authenticated and will not have access to the user’s data. But there is another, subtler way this process can be used for nefarious purposes.

The thing is, you can (and should) expect public keys to be just that — public. You should behave as if everyone on the internet knows your public keys. In fact, many services expose the public keys of their users. GitHub exposes them on their public API without any need for authentication; [here are mine](https://api.github.com/users/gmalette/keys). From a security perspective, that’s understood, and not a problem.

When you sign up to GitHub or Heroku, you will typically add your public key to your account. That allows cloning repos using urls such as "git@github.com/user/project". For convenience reasons, those services usually allow using more than one public key. Because the identification is based on the public key fingerprint, they will prevent two accounts from having the same public key, which can restrict the attack I’m about to describe, but not by much.

Let’s revisit the previous example, this time using public keys. Bob signs up for GitHub, and uses a key named "id_rsa". Bob then signs up to Heroku, using a different key named "id_rsa_heroku". Since SSH will by default always send the keys in the same order, we’ll pretend the key "id_rsa" is sent first. Let’s examine the SSH debug log when Bob connects to Heroku.

```
[1] debug1: Next authentication method: publickey
[2] debug1: Offering RSA public key: /Users/bob/.ssh/id_rsa
[3] debug1: Authentications that can continue: publickey
[4] debug1: Offering RSA public key: /Users/bob/.ssh/id_rsa_heroku
[5] debug1: Server accepts key: pkalg ssh-rsa blen 277
[6] debug1: Authentication succeeded (publickey).
```

The following happened:

- [2] — Bob offered his first key (id_rsa)
- [3] — the server tells Bob it doesn’t know the key; try another one
- [4] — Bob offers the second key (id_rsa_heroku)
- [5] — The server knows the key. It’s to Bob’s account. It produces a challenge
- [6] — authentication happens, and Bob is successfully logged in.

Alice knows all the keys, so she adds "id_rsa" to an empty Heroku account (remember, Bob uses "id_rsa_heroku"). What happens when Bob tries to connect to his Heroku account?

```
[1] debug1: Next authentication method: publickey
[2] debug1: Offering RSA public key: /Users/bob/.ssh/id_rsa
[3] debug1: Server accepts key: pkalg ssh-rsa blen 277
[4] debug1: Authentication succeeded (publickey).
```

The debug log of the SSH connection reads as follows:

- [2] — Bob’s SSH client sends the key "id_rsa"
- [3] — The server knows this key. It’s to Alice’s account. It produces a challenge
- [4] — Bob successfully responds to the challenge (he owns the private key)
- Bob is logged into Alice’s empty account
- Bob cannot access his own data

This can hardly be classified as an attack, yet Bob is not able to connect to his account. The message Bob will get is: "Your account alice@email.com does not have access to awesome-project", which will probably leave most users dumbfounded. This denial of service will cause headaches to advanced users, and will seriously ruin the day of less technical people. To be fair, Bob is not completely locked out. SSH allows specifying which key to use when connecting to specific servers.

A simple solution would be to avoid the single user login `git@service.com`, and use that as identification, for example `gmalette@service.com`.

To leave you with a simple takeaway, identification and authentication serve a fundamentally different purpose. Thinking they can be interchanged can lead to frustrating issues for users. Please understand what the difference is, and build systems accordingly.

Thanks to Lydia Krupp Hunter and Richard McGain.

[Comment or React](https://github.com/gmalette/gmalette.github.io/pull/5)
