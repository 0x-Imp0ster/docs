# Wireguard + Pi-Hole (docker) + VPS  

### This guide will set up a Wireguide VPN with Pi-Hole running in a docker container on a VPS.

## Prerequisites

1. A VPS, this guide assumes Ubuntu. Some steps may differ on other distro's
 - I recommend the cheapest Digital Ocean droplet (£5/month)
 - Docker (see this guide TODO)
2. Linux/mobile Client(s) to connect to the VPN


## The Plan

The VPS will only have ports 22/tcp and 51280/udp (wireguard) open externally.  Internally, the Pi-Hole containers ports 53, 80 and 443 will be mapped to the VPS Wireguard IP so that Pi-Hole is only accessible to clients connected to the VPN.
VPN clients will be configured to use Pi-Hole as the DNS server when connected to the VPN.

## Terminology

*WireGuard® is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography. It aims to be faster, simpler, leaner, and more useful than IPsec, while avoiding the massive headache. https://wireguard.com*
Peers, clients - interchangeable in this guide.  Wireguard works with 'peers' which are any hosts that are connected to the Wireguard VPN. Our VPN setup will be using the server/client model and so 'clients' is used to distinguish peers that are not the server.

### The configuration that we will end with

Host | Public IP | Wireguard IP | Private Key | Public Key
-----|-----------|--------------|-------------|-----------
VPS | 172.62.123.123 | 10.10.10.1 | 8GF9Y/uTIwbO3huAvR20basfEK97Q0hi2vGV4MEZMFc= | mSJyGg57UbaqGMHaCq7J5WTAbnhFqHBJWYd8ibmEamc=
Peer A | Dynamic | 10.10.10.10 | wA+Un72l07kJOKpO+csgncpdwJMbtnX5V29Y1Wcg+HU= | BkwYpR7w9tKhVEi66FfWqPHE9wg6MpQMS/ZKQuc6cRU=
Peer B | Dynamic | 10.10.10.11 | kNPqTmiSHCfiXRBXZqcaA0J34edECPuyc0GBr9r55VI= | RtGDli9VLP5MZODmDaCp7Ye6QiiinzNVucd7qEFZ3E0=
iphone | Dynamic | 10.10.10.12 | 0Kx+TmUSac7x2X2jIvsHlQgVwBeim/nlA5Lq7Rca2Xg= | 64F9Y+XYD5CS7D/WQlTSdcaU/6Pp3MArL4yWtIApGiI=

Becuase the Pi-Hole ports will be mapped to the hosts Wireguard IP, its IP address will effectively be `10.10.10.1`.

## Setting up the server

### Firewall rules

Using UFW we will set the default policy to block all incoming and allow all outgoing.

```bash
$ sudo ufw default allow outgoing
$ sudo ufw default deny incoming
```

Enable SSH so that we do not lock ourselves out before enabling the firewall

```bash
$ sudo ufw allow ssh
```

We will be using port `51280` for our Wireguard VPN so will open this now

```bash
$ sudo ufw allow 51280/udp
```

To enable the kernel to forward packets between interfaces, uncomment this line in `/etc/sysctl.conf`

```        
 net.ipv4.ip_forward=1
```

**Make sure you have allowed SSH before enabling the firewall!**  
```bash
$ sudo ufw enable
```

## Wireguard

### Install

#### Debian/Ubuntu

```bash
sudo apt update
sudo apt install wireguard
```

### Configure Wireguard as a VPN Server

Become root if not already
`$ sudo -i`

Generate private and public keys

```bash
cd /etc/wireguard
wg genkey | tee wg-server.key | wg pubkey > wg-server.pub
```

Example keys we will be using in this guide, **don't use the keys in this guide!** replace them with your own.

```bash
# cat wg-server.key
8GF9Y/uTIwbO3huAvR20basfEK97Q0hi2vGV4MEZMFc=
# cat wg-server.pub
mSJyGg57UbaqGMHaCq7J5WTAbnhFqHBJWYd8ibmEamc=
```

Set read-only permissions for the private key
```bash
chmod 600 wg-server.key
```

You can create any number of keys from any peer, for example you could generate keys for all the clients you plan to have by repeating the above command using different filenames.  Alternatively create the keys on each peer.

Generate keys for peers A, B and an iPhone

```bash
cd /etc/wireguard

wg genkey | tee wg-peer-a.key | wg pubkey > wg-peer-a.pub
wg genkey | tee wg-peer-b.key | wg pubkey > wg-peer-b.pub
wg genkey | tee wg-peer-iphone.key | wg pubkey > wg-peer-iphone.pub

chmod 600 ./wg-peer*.key
```


### Create the server configuration file

A configuration file is made up of an Interface section relating to the Wireguard interface for the host and one or more Peer sections that relate to the hosts it will connect to.

Create the configuration file in `/etc/wireguard` and call it `wg0.conf` where `wg0` is the name you want the interface to have.

#### Interface Section


```ini
[Interface]
PrivateKey = 8GF9Y/uTIwbO3huAvR20basfEK97Q0hi2vGV4MEZMFc=
Address = 10.10.10.1/24
ListenPort = 51280
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
SaveConfig = true
```

**PrivateKey:**  The contents of the private key file.  
**Address:** This can be any address you choose but the peers will need to be within the same subnet
**ListenPort:** Again, this can be anything
**PostUp and PostDown:**
PostUp creates the rules when the `wg0` interface is created and PostDown removes them when `wg0` is closed down.
These rules Forward traffic to and from the Wireguard interface to the hosts Internet-facing interface.  The MASQUERADE rule will change the source/destination addresses of the packets to match the interfaces accordingly.  In other words, VPN clients will have a source address of the VPN server. 
**SaveConfig:** If true, the configuration file (wg0.conf) is overwritten with the current state of the interface before shutdown.  See `man wq-quick`.  

#### Peer Sections

To configure the peer sections of the servers `wg0.conf` you will need the **peers public key** and its Wireguard IP.  For simplicity, we will use the keys we generated earlier on the server.

Add the peer section to `wg0.conf` on the server like so:

```
[Peer]
# Peer A
PublicKey = BkwYpR7w9tKhVEi66FfWqPHE9wg6MpQMS/ZKQuc6cRU=
AllowedIPs = 10.10.10.10/32
```

**# Peer A:** Just a comment to help keep track of which peer this is.  
**PublicKey:** This is the public key of the peer.  
**AllowedIPs:** This is a list of IPs with CIDR masks that defines where incoming traffic is allowed from this peer and where outgoing traffic is directed.  So for VPN clients we restrict it to a /32 and the IP we put here will be used in the `Address` field of Peer A's Interface section.  This IP can be anything that falls in the subnet of the `Address` in the Interface section we entered above.

Repeat this process for all required peers.

Our example `wg0.conf` for the VPN server now looks like this

```
[Interface]
PrivateKey = KGo9ksNn1NUzugoOOJxcJmBPR6pmr9r+6fCpW8sYL3Y=
Address = 10.10.10.1/24
ListenPort = 51280
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
SaveConfig = true

[Peer]
# Peer A
PublicKey = BkwYpR7w9tKhVEi66FfWqPHE9wg6MpQMS/ZKQuc6cRU=
AllowedIPs = 10.10.10.10/32

[Peer]
# Peer B
PublicKey = RtGDli9VLP5MZODmDaCp7Ye6QiiinzNVucd7qEFZ3E0=
AllowedIPs = 10.10.10.11/32

[Peer]
# iphone
PublicKey = 64F9Y+XYD5CS7D/WQlTSdcaU/6Pp3MArL4yWtIApGiI=
AllowedIPs = 10.10.10.12/32
```

The server is now ready but we will start it later.

## Setting up the client

### Install

#### Debian/Ubuntu based distro's

`$ sudo apt update`
`$ sudo apt install wireguard`

##### Arch/Manjaro

`$ sudo pacman -Syyu`
`$ sudo pacman -Syu wireguard-tools`

#### Configuration file

The client configuration files also need an Interface and Peer sections but will have different fields.

Either copy the private and public keys from the server if created there or generate new ones on the client as shown above.

As with the server config file, save this file in `/etc/wireguard` and call it `wg0.conf` where `wg0` is the name you want the interface to have.

#### Interface Section

```
[Interface]
# Peer A
PrivateKey = wA+Un72l07kJOKpO+csgncpdwJMbtnX5V29Y1Wcg+HU=
Address = 10.10.10.10/24
DNS = 10.10.10.1
```

**# Peer A:** A comment
**PrivateKey:** The contents of the **private key** for peer A.
**Address:** This will be the private VPN IP and needs to match the address specified in the `AllowedIPs` field for Peer A in the servers `wg0.conf`
**DNS:** We will be using Pi-Hole as the DNS server.  Pi-Hole will be running on the same VPS as the Wireguard server and be configured to only be accessible to VPN clients.

#### Peer Section

The peer for the client will be the VPN server.

```
[Peer]
# Server
Endpoint = 178.62.123.123:51280
PublicKey = mSJyGg57UbaqGMHaCq7J5WTAbnhFqHBJWYd8ibmEamc=
AllowedIPs = 10.10.10.1/32, 0.0.0.0/0, ::/0
PersistenKeepAlive = 25
```
**Endpoint:** The hostname or public IP address of the VPS and the port it is listening on  
**PublicKey:** The public key of the VPN server
**AllowedIPs:** The private IP for the VPN server as a /32. Adding 0.0.0.0/0 will route all traffic through the `wg0` interface to the peer (the VPN server).  `::/0` is the shorthand for all IPv6 traffic (optional, you should know if you are using IPv6).
**PersistentKeepAlive:** Send an empty Wireguard encrypted packet every _n_ seconds to keep the connection alive (optional). 

Repeat this process for all peers.

The full config for Peer A will look like this:

```
[Interface]
# Peer A
PrivateKey = wA+Un72l07kJOKpO+csgncpdwJMbtnX5V29Y1Wcg+HU=
Address = 10.10.10.10/24
DNS = 10.10.10.1

[Peer]
# Server
Endpoint = 178.62.123.123:51280
PublicKey = mSJyGg57UbaqGMHaCq7J5WTAbnhFqHBJWYd8ibmEamc=
AllowedIPs = 10.10.10.1/32, 0.0.0.0/0, ::/0
PersistenKeepAlive = 25
```

Do not start Wireguard yet because we have set a DNS server, DNS will fail until we have set up Pi-Hole.

Repeat this on all other VPN clients.

For mobile clients using the Wireguard app you can import the config file using a QR code.

The following steps can be done on either the server or another Linux system.

1. Generate keys for the mobile client
2. Create a config similar to the one above, replacing the Private key and changing the address.
3. Install qrencode
  - Ubuntu: `apt install qrencode`
  - Arch/Manjaro: `pacman -Syu qrencode` 
4. Run this command to generate the QR code:
`qrencode -t ansiutf8 -r wg-iphone.conf`


Scan the QR code using the Wireguard app


### Pi-Hole

Running Pi-Hole in a docker container works really well and allows it to be easily set up and teared down in seconds while retaining its logs if wanted.  The official container comes with an easy bash script to set it up but needs a little tweaking to work with our Wireguard VPN.

The official Pi-Hole docker container can be found [here](https://hub.docker.com/r/pihole/pihole) and the corresponding set up script [here](https://github.com/pi-hole/docker-pi-hole/blob/master/docker_run.sh), shown below with the changes.  

```bash
#!/bin/bash

# https://github.com/pi-hole/docker-pi-hole/blob/master/README.md

WG_SERVER_IP="10.10.10.1"
docker run -d \
    --name pihole \
    -p $WG_SERVER_IP:53:53/tcp -p $WG_SERVER_IP:53:53/udp \
    -p $WG_SERVER_IP:80:80 \
    -p $WG_SERVER_IP:443:443 \
    -e TZ="Europe/London" \
    -v "$(pwd)/etc-pihole/:/etc/pihole/" \
    -v "$(pwd)/etc-dnsmasq.d/:/etc/dnsmasq.d/" \
    --dns=127.0.0.1 --dns=1.1.1.1 \
    --restart=unless-stopped \
    --hostname pi.hole \
    -e VIRTUAL_HOST="pi.hole" \
    -e PROXY_LOCATION="pi.hole" \
    -e ServerIP="127.0.0.1" \
    --add-host peer-a:10.10.10.10 \
    --add-host peer-b:10.10.10.11 \
    --add-host iphone:10.10.10.12 \
    pihole/pihole:latest

printf 'Starting up pihole container '
for i in $(seq 1 20); do
    if [ "$(docker inspect -f "{{.State.Health.Status}}" pihole)" == "healthy" ] ; then
        printf ' OK'
        echo -e "\n$(docker logs pihole 2> /dev/null | grep 'password:') for your pi-hole: https://${IP}/admin/"
        exit 0
    else
        sleep 3
        printf '.'
    fi

    if [ $i -eq 20 ] ; then
        echo -e "\nTimed out waiting for Pi-hole start, consult check your container logs for more info (\`docker logs pihole\`)"
        exit 1
    fi
done;
```

1. Change the WG_SERVER_IP value which will map the ports to use the VPN servers Wireguard IP - line 5
2. Change the timezone - line 11
3. Add peer hostnames - line 20+.
Because we aren't using the Pi-Hole as a DHCP server, it will list the clients using their IP address.  The `--add-host` parameter will add these hosts to the `/etc/hosts` file within the Pi-Hole container.
When you add new peers to the VPN, simply add them to this script, stop and remove the container and re-run this script.  All the data will be restored because of the volumes mapped to `etc-pihole` and `etc-dnsmasq.d`.
4. Rename the script and make it executable
 a. `mv docker_run.sh pi-hole-docker.sh`
 b. `chmod +x pi-hole-docker.sh`
5. Run the script

```bash
root@wg-pihole:~# ./pi-hole-docker.sh 
WARNING: Localhost DNS setting (--dns=127.0.0.1) may fail in containers.
Unable to find image 'pihole/pihole:latest' locally
latest: Pulling from pihole/pihole
7d2977b12acb: Pull complete 
78cf089748c9: Pull complete 
efe3d34b3c25: Pull complete 
bc429e9f6f2d: Pull complete 
0681d08f984c: Pull complete 
bd2f3a8bdba3: Pull complete 
27b593f694e7: Pull complete 
7c7778116f7e: Pull complete 
350c3c0ad115: Pull complete 
359b49d1a603: Pull complete 
882d1eb3c4fe: Pull complete 
a6297928222b: Pull complete 
dcede3de7fc3: Pull complete 
Digest: sha256:f26dc1beaec171b2e53e107643fb2ea7dfbb0afea0e9572671d6737b986a8870
Status: Downloaded newer image for pihole/pihole:latest
0b40f0cacba81743f4df0d4f770f523c50154cbaf9ec1affee737cad2dded910
Starting up pihole container .......... OK
Assigning random password: f6T7XHfS for your pi-hole: https:///admin/
```

**Note** the generated password at the end of the output.  If this is the first time running the script then change the password as shown below.  If you are re-running the script without deleting the `etc-pihole` directory the password should still be the same as you had previously.

**Change the Pi-Hole admin password**

```bash
# root@wg-pihole:~# docker container exec -it <container_name> <command> <arguments>

root@wg-pihole:~# docker container exec -it pihole pihole -a -p
Enter New Password (Blank for no password): 
Confirm Password: 
  [✓] New password set
root@wg-pihole:~#
```

### Startup Wireguard

Now that Pi-Hole is running, it is safe to start up Wireguard.

Wireguard comes with the `wg-quick` script which is a wrapper for the `wg` and `ip` commands to set up and close down the Wireguard interface based on the settings supplied in a config file.  The config file needs to have the filename of the interface name with the extension of `.conf` and be located in `/etc/wireguard` directory.  You can alternatively supply a path to a different config file. See `man wg-quick` for more.

To start Wireguard using wg-quick where the configuration file we created is `/etc/wireguard/wg0.conf`:

```bash
root@wg-pihole:~# wg-quick up wg0
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.10.10.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Start Wireguard on the peers in the same way

To see the status of the connections simply run `wg` with no arguments as root or using sudo.

```bash
[nath@peer-a ~]$ sudo wg
interface: wg0
  public key: BkwYpR7w9tKhVEi66FfWqPHE9wg6MpQMS/ZKQuc6cRU=
  private key: (hidden)
  listening port: 33742
  fwmark: 0xca6c

peer: mSJyGg57UbaqGMHaCq7J5WTAbnhFqHBJWYd8ibmEamc=
  endpoint: 178.62.123.123:51280
  allowed ips: 10.10.1.1/32, 0.0.0.0/0
  latest handshake: 7 seconds ago
  transfer: 6.06 KiB received, 17.71 KiB sent
  persistent keepalive: every 25 seconds
```

The interface can be managed by `systemd` to start automatically by enabling it as a service:

`$ sudo systemctl enable wg-quick@wg0.service`


### Accessing Pi-Hole

Once connected to the VPN you can access Pi-Hole from http://10.10.10.1/admin.
To get to Pi-Hole using the hostname `p.hole/admin` add it to your `/etc/hosts` file:
`10.10.10.1		pi.hole`

After logging in, the dashboard should show the clients by host-name that you included in the setup script.

