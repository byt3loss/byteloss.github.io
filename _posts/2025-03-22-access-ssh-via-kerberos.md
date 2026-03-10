---
layout: post
title: Access SSH via Kerberos
date: 2025-03-22
categories: [blog]
tags: [network, post-exploitation]
---

*Legit (and dirty) tricks to access a remote server via SSH when is required a Kerberos ticket.*

![](/assets/images/cerberus.jpeg)

During a CTF recently played, I faced a target exposing an SSH service but unlike usual servers that authenticate users via private keys or passwords, it was asking for a Kerberos ticket. During the process, I learned a thing or two that I'd like to write down to share the knowledge and spare others in the CTF community — or my future self — from a tough hour.

I won't dig too much in what Kerberos is and how it works, because it is out of scope, but I suggest an article called [Kerberos made easy](https://www.vanimpe.eu/2017/05/26/kerberos-made-easy/) for those who want to explore further.

## Interacting with Kerberos on Linux

Before interacting with Kerberos, we are going to check if Kerberos is actually available on the target (ex. target.htb) with a port scan, hopefully gathering some more information too.

```sh
nmap target.htb -p 88 -sV -sC
```

```
PORT   STATE SERVICE      VERSION
88/tcp open  kerberos-sec Microsoft Windows Kerberos (server time: 2025-03-22 21:02:19Z)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

The service is running on the default port 88, and the scan also retrieved the server time, which is very valuable since Kerberos can be susceptible to time differences.

### Obtaining a ticket

After some reconing we found a vulnerability on the website, which let us execute a reverse shell and gain a foothold on the target. Looking around the website source code, we came across DB credentials, accessed the application database, stole the usernames and the hashes, cracked the hashes offline and successfully retrieved an admin password. We are now going to try those credentials on the target, authenticating via SSH.

```sh
$ ssh user@target.htb

user@target.htb: Permission denied (gssapi-with-mic,keyboard-interactive).
```

The server denies our connection and informs us that the two methods accepted are:

- **GSSAPI-with-MIC** (Generic Security Services API with Message Integrity Code): Kerberos-based, secure, passwordless authentication.
- **Keyboard-Interactive**: a flexible challenge-response method, often used for MFA.

Ok, so this means that first of all we must authenticate to Kerberos, obtain a valid ticket and then try to authenticate to SSH again.

### Installing and configuring krb5-user

If it's not already installed, install it via apt (Ubuntu/Debian distros).

```sh
sudo apt install krb5-user
```

Once installed, apt will spawn a cli wizard where you should enter the default domain. You can skip those steps and edit the file `/etc/krb5.conf` later. The following is my configuration.

```ini
[libdefaults]
    default_realm = TARGET.HTB
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    forwardable = yes

[realms]
    TARGET.HTB = {
        kdc = target.htb
        admin_server =  target.htb
        default_domain = target.htb
    }

[domain_realm]
    .target.htb = TARGET.HTB
    target.htb = TARGET.HTB
```

At this point we are ready to obtain the ticket.

```sh
echo "Password123" | kinit -l 1000 user@TARGET.HTB
```

I set the lifetime of the ticket to 1000 seconds (`-l 1000`), sometimes Kerberos servers don't enforce a maximum lifetime.

Running `klist` you should be able to see some info about the ticket and that it stored in `/tmp/krb5cc_1000`.

## SSH Connection

### SSH auth - Clock skew too great

Trying to authenticate now via SSH we are still getting errors, using the `-v` flag for a verbose output we get the following error message.

    debug1: Next authentication method: gssapi-with-mic
    debug1: Unspecified GSS failure.  Minor code may provide more information
    Clock skew too great

Do you remember when, after the port scan, I mentioned that the retrieved server date and time is valuable information?

The server is in a different timezone, and we should align our system's time with it.

At this point I tried to stop and disable services, set ntp to false, etc (I'm on Kali) but there was no way to prevent my machine to adjust the date after changing it with the command `date`, so I went for the dirtiest but most effective way: spawning a root shell and executing a while true loop to update the clock every second.

```sh
while true; do date -u -s "2025-03-21 17:55:00Z"; sleep 1; done
```

### SSH auth - Server not found in Kerberos database

One more time, we try to authenticate via SSH but...

    debug1: Next authentication method: gssapi-with-mic
    debug1: Unspecified GSS failure.  Minor code may provide more information
    Server not found in Kerberos database

Don't worry, I got you. 

By default, when using GSSAPI authentication in SSH, OpenSSH derives the service principal name (SPN) as follows: `host/<hostname>@<realm>`.

If you do not specify the flag `-oGSSAPIServerIdentity`, SSH will attempt to use: `host/<target_hostname>@<KERBEROS_REALM>`.

- `<target_hostname>` → The hostname you specify in the SSH command.
- `<KERBEROS_REALM>` → Taken from your Kerberos configuration (/etc/krb5.conf).

So if you run `ssh user@target.htb`, SSH will try to authenticate with the principal `host/target@TARGET.HTB`.

In case it is different, you need to specify it with `-oGSSAPIServerIdentity=<target_hostname>`. In my case, the command has become:

```sh
ssh user@target.htb -oGSSAPIServerIdentity=dc.target.htb
```

This time it should authenticate you without any more problem... Unless the ticket has expired in the meantime!

## Conclusion

I hope this brief article has been helpful! If you want to try the CTF that inspired it, it is available on HackTheBox: [TheFrizz](https://app.hackthebox.com/machines/652).