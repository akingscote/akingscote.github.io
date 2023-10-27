---
title: "Information Security Fundamentals"
date: "2017-04-05"
categories: 
  - "security"
coverImage: "Featured-Image.jpg"
---

In this post, I’ll try to simplify some of the terminology and concepts within information security. This post will just cover some of the basics, but even the basics can be pretty exhaustive depending on your definition of basic.

# CIA Triad

The term CIA triad refers to the balance of Confidentiality, Integrity and Availability within information security. Similar to the [fire triangle](http://www.sc.edu/ehs/modules/Fire/01_triangle.htm),  all elements must be present in order for the aim to be achieved of having a secure system. It is one of many security policies to help ensure people are thinking about vulnerabilities within their infrastructures.

![](images/CIA-Triad.png)

# Confidentiality

Confidentiality means ensuring data cannot be read by anyone unintended. This is commonly referred to as data encryption. It is basically mashing up the data so much that it cannot be read unless you know the key to unlock it. The way I like to visualise it is kind of like a Rubiks cube, you need to know the sequence (key) to unlock it. This is an extremly simplifeid analogy but should help you grasp the basics.

There are three commonly used encryption methods:

DES – Data Encryption Standard

3DES – Triple Data Encryption Standard

AES – Advanced Encryption Standard

Everytime you encrypt data, the outcome is different which makes it very difficult to reverse engineer. With the right key, you can decrpt the mashed up information into its original format. DES is now obsoleted because somebody cracked the “key” – the combination to the metaphorical Rubiks cube. However AES is still very secure and widely implemented and DES is still doing well.

# Integrity

Integrity means ensuring data has not been tampered with. As data travels from A to B, there may be an opportunity for somebody to intecept the message and change what is being sent. Another metaphor would be Chinese whispers. Ideally, the message wont have changed, but can you trust the class joker not to change it?

Common methods to ensure integrity are:

_Encryption_ – using asymmetric cryptography (more on this later), we can ensure that nobody has changed any data as only owner of the private keys can encrypt/decrypt.

_Anti-Replay_ – within IPSec (a network security protocol), when a security association has been established between two parties, they both set their “counters” to zero. Each packet sent from A-B will have a sequence number (number 1 for the first packet being sent, number 2 for the second packet and so on…) Each time a packet is sent, the receiver checks that the number and keeps track of what it has received. If a hacker were to somehow add in some data, the counters between A & B would not be in synchronization and the packet would be discarded. Higher level protocols (OSI layer 4 (transport) as opposed to layer 3 (network) where we are now) would resolve any packet loss to ensure reliable delivery of all data.

It is possible to achieve data integrity via hashing although its not nearly as effective as encryption

# Availability

Availability means ensuring that information is readily accessible to authorized parties.  Some hacks have the intention of simply causing disruption to a service. For example, if Yahoo DDOS’ed Google and google somehow became unavailable, then Yahoo might become more popular during that downtime. Availability is extremely important to businesses and many have a policy of 99% availability across the year. This means that only 1% of time a year is allowed where systems are offline (usually for maintenance or upgrades)

Availability can be improved by a number of methods:

- Ensure system resources are reliable and well maintained
- Ensure applications and systems are updated to latest verisons
- Architecture has redundancy
- Load balanced topology to reduce strain on a single resource (reducing single point of failures)
- UPS – Unteruptable Power Supply which starts in the event of a power failure ensuring systems say online
- Implement firewalls to block unwanted traffic
- Correctly and efficient system configuration (effective routing, avoiding broadcast storms)
- Physical security on devices – this isnt just limited to availability. Servers and network devices should be stored in a locked environment under surveillance.

There are many more ways that availability can be achieved, hopefully this gives you an idea to some of the methods possible.

# Hashing & Encryption Explained

## Hashing

Hashing is a mechanism for securely storing data, namely passwords. It is an _“irreversible”_  method for obsucring data so that it is not stored in plain test.  When a hashing algorithm is applied to data, the result is always the same. Hashing converts data into a fixed length. For example, if i were to hash the letter ‘a’ and the word ‘hello’ they would both be the same hash size even though ‘hello’ is longer than the letter ‘a’.

There are two popular hashing protocols:

- MD5 – Message Digest 5
- SHA – Secure Hashing Algorithm

The way that hashing can contribute to integrity is by examining the hashed message before and after transmission to see if there are any changes. As hashing is one-way, fixed length & irreversible (some can be cracked using hash tables), there will definitely be a difference in the sent hashed message and the received hash message if the message has been tampered with.

For example, if I want to encrypt the password “nerdgrad” lets say the hash result looks like this 394u2rjfd83j9d39jd9j3d3a. Nerdgrad is a pretty human readable password, there are tools out there where you could pass human readable words (dictionary attack) into a hash generator and try to get the same outcome. If you do, then you’ve cracked the password!

A way to make a hash more secure it to add a ‘salt’ to the password before it is hashed. A salt means to add a string of numbers & letters to your password before you hash it. So lets say “nerdgrad” becomes “nerdgrad74yht74h983d” which is then hashed to create “8fghs8ahf8h38h3d838hdhd38hd3”. The longer the salt, the more difficult it is to crack your password because nobody is going to guess nerdgrad74yht74h983d as a password!

Whenever your password is stored anywhere (like on say facebooks database), they **never** store your password plaintext. It would be your hashed password. Even if their database got compromised, the hacker still needs to somehow try and reverse the hash and remove the salt.

A good way to visualise hashing is to try the following exercise:

1. Pick an actual word for a password (one that is on Wikipedia or in the Oxford dictionary).
2. Go to [miraclesalad’s hash generator](http://www.miraclesalad.com/webtools/md5.php) and apply the MD5 hashing algorithm onto your password.

![](/images/md5-hash-300x269.jpg)

![](/images/md5-hash2-300x212.jpg)

Notice that ‘stationary’ and ‘a’ generate the length size hash – hashes are always the same length.

Now hashing isn’t meant to be reversed, if you use the same hashing algorithm, the result will always be the same. So when you log into Facebook and type in your password, they will apply a hash to your password and if the result matches the hashed value that they have stored in their database as your password. If it matches, then it will let you log in.

That being said, there are some tools than **can** reverse hashes. These are called rainbow tables or hash tables.

Take your MD5 hash from before and place it into the [Hash Cracker tool on crackstation.net](https://crackstation.net/)

![](/images/reverse-hash-300x189.jpg)

Notice that there has been a match and the program has been able to un-hash my password and figure out that it was ‘stationary’.

Now, if we go back to the miraclesalad’s MD5 tool and add in a salt to our password. Just add the following made up salt – ‘h3ujIU3dsm3’.

In reality they are properly generated and complicated but ive just made this one up.

![](/images/md5_salted_hash-300x241.jpg)

Now take that new hash and put it in crackstations tool to try and get the password.

![](/images/md5_salted_unhash-300x194.jpg)

Youll notice that you are unable to crack the password! So hopefully that gives an idea as to what hashing is and how it can be used with a SALT to improve security. The longer the salt, the harder it is to guess and un-hash.

## Encryption

As mentioned, encryption is about ensuring confidentially in data – that only intended parties can view the data. Within cryptography, there are the concepts of Symmetric and Asymmetric key pairs.

## Symmetric Keys

With symmetric-key encryption, the same key is used for both encryption and decryption by both parties. When establishing a connection between peers using symmetric key encryption, the key can be sent in plaintext or sent to the other party by other means. For example, I want to establish connection with a friends server down the road. I set the key to be “nerdgrad”. Now I could either manually walk down the road, go to his server and configure his server to accept my key of “nerdgrad” or I could text him it and get him to do it. The point is that either end needs to be configured with the same key.

![](/images/Symmetric-Key-300x75.jpg)

In the diagram above, the text “hello” has been encrypted with a key. The message just turns to unreadable jargon.

The destination (server B) receives the encrypted message and applied the configured key to the message. This decrypts the message and the original message is extracted.

This is a grossly over simplified explanation, if you want to know the details you can read more [here](http://www.cs.cornell.edu/courses/cs5430/2010sp/TL03.symmetric.html) and

## Asymmetric Keys

With asymmetric keys, two keys are created that have a special relationship. Whatever you encrypt with one of the keys, you decrypt with the other and vise versa. One of these keys is openly distributed, this is called a public key. The other is kept secret on the server/router whatever and is never distributed, this is called a private key.

If you encrypt with the private key, then anybody could decrypt your message because they could decrypt it with the openly distributed public key. This does however have some benefit because the receiver of the message will know that the message is genuine. It must have come from the correct server (not spoofed) because it **has** to be encrypted with the private key which only the server has access too.

If you encrypt with the public key, then only the intended party can decyrpt the message. This is great, however the message could have come from anyone because it would have been encrypted with the public key and anybody has access to that.

![](/images/Asymmetric-Keys-300x149.jpg)

The solution is to do both – firstly encrypt with the private key which will ensure that the message is genuine, and to then encrypt again with the destinations public key (not your public key, but theirs). That way the destination will be the only person that can decrypt the message and they will know it is legitimate.

# Authentication, Authorization & Accounting (AAA)

AAA is a method for dynamically controlling access to endpoints & resources within a network and provide a detailed account of who did what and when.

Authentication is a way of identifying where a message came from and determining if they are who they say they are. This is necessary to a avoid spoofed messages where somebody pretends to be someone they are not.

Authorization is validation if a sender has been granted access to a particular resource.

Accounting is keeping account of who did what and when; for example an audit trail.

Methods to achieve AAA include:

- Asymmetric Cryptography – authentication
- PAP and CHAP network protocols – authentication
- RADIUS Servers – dedicated server to achieve AAA
