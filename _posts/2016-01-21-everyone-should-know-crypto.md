---
layout: post
title: Everyone should know cryptography
categories: [security, cryptography]
tags: [security, cryptography]
published: True

---
Everyone should know cryptography.

I don't mean "know cryptography" like knowing the mathematical nitty-gritty or academic research of the month. Everyone should know cryptography _as a tool_. Like how everyone knows how to _use_ a screwdriver, but not necessarily how to build a factory that manufactures screwdrivers.

Here's why.

## Why do I need cryptography?

### The parable of the bank account

Let me tell you a (fictional) story of when I tried to open a bank account with _Banks R Us_.

I call up the number from the _Banks R Us_ ad, and Bob from _Banks R Us_ answers.

_Bob_: Bob from Banks R Us here.

_Me_: I'd like to open a bank account at _Banks R Us_, please.

_Bob_: Great! I'll just need your social security number to open the account.

That sounds fine--I trust _Banks R Us_ and their friendly business brand (the backwards _R_ on their logo really appeals to the kid in me), and Bob seems like a nice guy. So I'm okay with telling him such a secret piece of information.

_Me_: Okay, my social security number is--

_Bob_: Wait! That's not how this works.

Bob tells me that I can't just give my social security number over the phone. I need to write the number on a piece of paper and give it to my postman. Then the postman will bring it to my local gym and drop it off to whomever is working the front desk. Then a courier from _Banks R Us_ will pick it up and drop it in Bob's inbox.

_Me_: That's insane! Any one of those people could steal my secret, and I wouldn't even know if they did! I'm hanging up.

Was I right to hang up?

### The internet was designed by Bob

This story is a heavy-handed illustration of how insane passing around secrets over the internet is.

It's fairly obvious what is wrong with Bob's scheme: every person in the long chain has the ability to steal my secret social security number!

Unfortunately, the internet was designed by Bob. Communication over the internet is passed between many parties--just like in the complicated scheme Bob asked for.

Of course, this isn't just restricted to social security numbers. All of the following internet traffic is exposed, as well:

- browsing history
- passwords
- PINs
- birthdate
- credit card number

### Wait, that can't be right!

Of course, if it were that easy, everyone who's ever bought anything online with a credit card would have gotten the information stolen, and e-commerce would cease to exist.

The reason that doesn't happen is because of smart use of *cryptography*, which allows for encoding of messages through the internet so that only intended parties can read the message.

This is why _cryptography_ is important.

## Why do I need to _understand_ cryptography?

Encryption cannot protect you if you don't understand how to use it.

### The parable of the idiot zoo

Let's say you go to the local zoo, _Zootastic_. You head straight to the gorilla cages (because awesome), and you see this.

![unlocked padlock]({{ site.url }}/assets/unlocked.jpg)

You understand how a lock works. And that lock is clearly not doing anything. This zoo is clearly unsafe. You quickly leave _Zootastic_, and even alert the zookeepers.

### The internet was built by folks from _Zootastic_

Yet another heavy handed illustration, sorry.

Not every part of the internet out there is a disaster like _Zootastic_, but [a lot of them are]().

If you do not know how to recognize when cryptography is protecting you and when it isn't, you'll find yourself in a dangerous situation without even realizing it.

### _You_ might be from _Zootastic_

The folks at _Zootastic_ failed because they did not understand how to secure something that was important to secure. But before laughing at those guys, let's not forget that _we might be them_.

You may need to secure something, but not know how to secure it. Or maybe you think you know how to secure it but fail to do it properly, like _Zootastic_.

This is why _understanding cryptography_ is important.

## What should I do?

I'm going to be writing a short series on some basic crypto concepts and how to apply them. Check back later for updates!

Additionally, there are tons of excellent resources out there. The internet, a crazy _Zootastic_ of a place though it is, has a wealth of information for you, if you just look for it.
