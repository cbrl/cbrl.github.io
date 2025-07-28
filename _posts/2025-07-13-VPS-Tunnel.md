---
title: Simple Point-To-Point VPS Tunnel
date: 2025-07-13 11:03:23 +/-0500
categories: [Networking]
tags: [vps, wireguard, nginx, tunnel, reverse proxy]
---

## Overview

Many people who self-host services may want to expose them to the internet, but also might have
issues with NAT, or prefer to avoid exposing their server's IP and opening ports on their firewall.

[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
is one solution that you'll almost certainly come across in this situation. It's easy to set up and
works very well, but it does have some drawbacks. Cloudflare imposes a fairly restrictive
[upload limit](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#upload-limits),
and if you're self-hosting your services there's a good chance you might not prefer to insert an
SSL-terminating third party into your stack. It also only supports HTTP traffic, so you're out of
luck if you want to proxy a service that doesn't work over HTTP.

The common alternative, and the focus of this post, is to run a similar tunneling setup through your
own VPS.

In my case, I have a set of services with a reverse proxy and authentication configured on my home
network, and want a simple tunnel that I can point my domain to and have all traffic forwarded to
the home network.

![VPS Tunnel Concept](/assets/posts/2025-07-13-VPS-Tunnel/VPSTunnelConcept.drawio.svg)

## VPN Tunnel Solutions

There are several commonly cited services that can be used to create this kind of tunneled setup,
such as:

- [Pangolin](https://github.com/fosrl/pangolin)
- [Tailscale](https://tailscale.com/)
- [Netmaker](https://www.netmaker.io/)
- [ZeroTier](https://www.zerotier.com/)

Pangolin is a good all-in-one solution, integrating the VPN tunnel and reverse proxy into a single
product with a fancy UI. However, a lot of its functionality is unnecessary if you already have
reverse proxy and authentication solutions configured, or just prefer to manage them separately.
Even for the more narrowly focused VPN tunnels listed above, a lot of their complexity isn't
required for a simple point-to-point connection.

For scenarios like this it's actually extremely easy to manually configure a tunnel, so much so
that there isn't much motivation to opt for higher level abstractions like Tailscale or Pangolin.
The manual option can also give a bit of clarity for someone new to these concepts, who doesn't
necessarily have the intuition about which pieces are required and how they all work together.

## Simple Tunnel With Wireguard + Nginx

Wireguard is the underlying VPN tunnel that several of the services listed above also rely on. In
the steps below, we'll use it to create a VPN with two peers: the VPS and reverse proxy server.
Once this is completed, we simply need to forward traffic from the VPS to the reverse proxy over
the wireguard network. This is generally pretty easy to set up with any reverse proxy that supports
the PROXY protocol. In our case we'll use Nginx.

![VPS Tunnel](/assets/posts/2025-07-13-VPS-Tunnel/VPSTunnel.drawio.svg)

The material below assumes that you have already have a VPS and that your domain's DNS records are
pointing to it.

### Wireguard

Before worrying about routing different types of traffic between devices, let's set up a simple
wireguard connection between them.

First, install wireguard on both devices:

```bash
sudo apt update
sudo apt install wireguard
```

Wireguard uses asymmetric cryptography, so we'll need to generate a public/private key pair for
each device. Navigate to `/etc/wireguard` and create the keys like so:

```bash
wg genkey | (umask 077 && tee privatekey) | wg pubkey > publickey
```

> These `privatekey` and `publickey` files aren't used by wireguard. Their contents will go in the
> config files that we're about to create, but they're good to keep for reference.
{: .prompt-tip }

Now we need to create the configuration files on each device at `/etc/wireguard/wg0.conf`. The
configuration parameters will be explained further in the next section

First create the wireguard configuration on the VPS:

```ini
[Interface]
PrivateKey = <VPS_PRIVATE_KEY>
Address = 10.0.0.1
ListenPort = 51820

[Peer]
PublicKey = <HOME_SERVER_PUBLIC_KEY>
AllowedIPs = 10.0.0.2
```
{: file='/etc/wireguard/wg0.conf'}

And then one on the home server:

```ini
[Interface]
PrivateKey = <HOME_SERVER_PRIVATE_KEY>
Address = 10.0.0.2

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820
AllowedIPs = 10.0.0.1
PersistentKeepalive = 25
```
{: file='/etc/wireguard/wg0.conf'}

> The file name doesn't have to be `wg0`. If you'd like to use a different name, just replace any
> usage of `wg0` in this post with your chosen identifier.
{: .prompt-tip }

And that's the entire configuration process. On each device, run `sudo wg-quick up wg0` to start 
wireguard. You should be able to ping the other device (e.g. `ping 10.0.0.1` from the home server).
If you can't ping the other device, check your peer endpoint and firewall settings.

```console
user@machine:~$ ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=38.9 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=40.0 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=38.5 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=42.6 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 38.490/40.001/42.590/1.597 ms
```

Instead of manually starting the wireguard interface, you can enable a background service that will
automatically start with your system.

```bash
sudo systemctl enable wg-quick@wg0.service
```

#### Wireguard Configuration Details

This section will discuss the basic parameters used above. These are far from comprehensive, but
are all you should need to achieve this goal. If you'd like to read detailed documentation for
wireguard, [this resource](https://docs.sweeting.me/s/wireguard) is excellent.

A basic wireguard configuration consists of an `[Interface]` section, which defines the local
machine's configuration, and a number of `[Peer]` sections, which define the other peers that will
connect to the VPN.

The typical `[Interface]` section will have the following parameters:

Name         | Description
-------------|------------
`PrivateKey` | The local machine's private key
`Address`    | The local machine's IP address on the wireguard network
`ListenPort` | The port on which the wireguard interface will listen for incoming traffic

The `ListenPort` variable is typically only specified on "server" machines that "clients" will be
connecting to. Wireguard will select a random port by default.

A peer configuration will have the following parameters:

Name                  | Description
----------------------|------------
`PublicKey`           | The peer's public key
`Endpoint`            | The public IP at which the peer is accessible
`AllowedIPs`          | The IPs that this peer will use for routing
`PersistentKeepalive` | The rate at which the local machine will ping the peer to keep the connection alive

The `Endpoint` can be left out for peers that don't have a stable endpoint or are behind NAT. It is
typically only specified for a machine that acts as a public server. Likewise, such a peer should
itself specify a value for `PersistentKeepalive` in order to ensure the connection with the remote
node is maintained.

### Nginx

Now that we have a VPN connection between the two devices, we need to forward traffic from the VPS
to the home server.

If you have some familiarity with networking, you might initially think of piping all traffic
through the wireguard interface using something like iptables. There are many tutorials out there
which describe this approach. However, this doesn't work well in all cases, and can be quite
difficult or tedious to debug given the lack of easily visible diagnostics. The primary issue with
this method is that it involves overwriting the source request's address with that of the VPS.
Losing the ability to distinguish clients on the other end end is not ideal, especially if you want
to run software like Crowdsec or fail2ban. Assuming your services are using HTTPS (as they always
should be), that software also wouldn't work on the VPS since you would just be passing encrypted
traffic that can't be inspected.

Nginx is one of several popular open source reverse proxy options that we can use to fix this
issue. Using the [PROXY Protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt),
reverse proxies can transmit additional information that preserves the originating client's
address instead of overwriting it with that of the VPS. The caveat is that the receiving end needs
to know that this protocol is being used. Pretty much any notable reverse proxy supports this
though.

As a bonus, Nginx will also allow us to forward UDP traffic if we desire (e.g. for a game server),
so this single solution will work for nearly any service running behind the tunnel.

Note that the reverse proxy running on your home server will need to be configured to receive the
PROXY Protocol headers and to trust proxied connections from your VPS's wireguard IP (i.e.
`10.0.0.1`). The steps to do this are specific to your choice of reverse proxy, so they won't be
covered here.

First, install nginx on the VPS:

```bash
sudo apt update
sudo apt install nginx
```

Edit the Nginx configuration file at `/etc/nginx/nginx.conf` and add the following block at the
top level.

```nginx
stream {
        server {
                listen 80;
                listen 443;
                proxy_pass 10.0.0.2:$server_port;
                proxy_protocol on;
        }
}
```
{: file='/etc/nginx/nginx.conf'}

> The default Nginx config may have a preset server exposing port 80 within the `http` block, which
> will conflict with the server definition above. Delete the `server` block inside the `http` block
> if this is the case.
{: .prompt-warning }

Now start the Nginx service, which should begin forwarding TCP traffic on ports 80 and 443 to the
home server (`10.0.0.2`) through the wireguard interface.

#### Nginx Configuration Details

The configuration here is fairly self-explanatory. The `stream` block is used for proxying raw TCP
connections instead of the more typical HTTP connection. Inside it we declare a service that
listens on port 80 and 443 and proxies communications to `10.0.0.2` on the same port, also passing
the PROXY Protocol header as well.

### Troubleshooting

If it isn't working, first make sure that the wireguard connection is active and the machines are
able to communicate with each other over wireguard. If this isn't working then it's likely a
configuration error or firewall issue.

If wireguard is working, you can use `tcpdump` to see if any HTTP traffic is flowing over the
wireguard interface:

```bash
sudo tcpdump -nn -i wg0 tcp and port 80 or port 443
```

If traffic is being proxied to the home server, then you may have an incorrect configuration on
your home server's reverse proxy. If there's no traffic, then you may have incorrectly configured
Nginx. [Enabling logging](https://docs.nginx.com/nginx/admin-guide/monitoring/logging/) in Nginx
may help diagnose this.

## Summary

This approach gives you a lightweight, transparent way to expose self-hosted services to the
internet, bypassing NAT and ISP restrictions, and without relying on third-party tunneling services
or complex mesh VPNs. The amount of configuration required for this simple point-to-point tunnel is
extremely minimal and requires little to no maintenance once functional.

## Additional Notes

### Oracle VPS

Oracle technically offers an entirely free VPS tier, without the requirement to even have a payment
method attached to your account. However, in practice it's extremely difficult to actually find a
time where they have resources available to provision.

If you upgrade your Free account to Pay As You Go (PAYG), you can usually provision a VPS with no
wait. This doesn't actually charge you any money as long as you use the Always Free tier of
resources.

Oracle's default VPS networking rules are fairly restrictive by default. You may need to edit your
Virtual Cloud Network's security rules to allow connections over ports other than 22. You also may
need to edit and reload the iptables rules at `/etc/iptables/rules.v4` to allow incoming
connections.
