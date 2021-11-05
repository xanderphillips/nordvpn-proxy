# nordvpn-proxy

[![lint dockerfile](https://github.com/edgd1er/nordvpn-proxy/actions/workflows/lint.yml/badge.svg?branch=main)](https://github.com/edgd1er/nordvpn-proxy/actions/workflows/lint.yml)

[![build multi-arch images](https://github.com/edgd1er/nordvpn-proxy/actions/workflows/buildPush.yml/badge.svg?branch=main)](https://github.com/edgd1er/nordvpn-proxy/actions/workflows/buildPush.yml)

![Docker Size](https://badgen.net/docker/size/edgd1er/nordvpn-proxy?icon=docker&label=Size)
![Docker Pulls](https://badgen.net/docker/pulls/edgd1er/nordvpn-proxy?icon=docker&label=Pulls)
![Docker Stars](https://badgen.net/docker/stars/edgd1er/nordvpn-proxy?icon=docker&label=Stars)
![ImageLayers](https://badgen.net/docker/layers/edgd1er/nordvpn-proxy?icon=docker&label=Layers)


This is a NordVPN client docker container using openvpn that use the recommended NordVPN servers, and opens a SOCKS5 (dante server) and http proxy (tinyproxy).

VPN servers selection is performed through nordnvpn API.(country, technology, protocol)

Added docker image version for raspberry.  

Whenever the connection is lost the unbound, tinyproxy and sock daemons are killed, disconnecting all active connections (tunnel down event).


## What is this?

This image is largely based on [jeroenslot/nordvpn-proxy](https://github.com/Joentje/nordvpn-proxy) with dante free socks server added. 
you can then expose port `1080` from the container to access the VPN connection via the SOCKS5 proxy.

To sum up, this container:
* Opens the best connection to NordVPN using openvpn and the conf downloaded using NordVpn API according to your criteria.
* Starts a dns server for container resolution
* Starts a SOCKS5 proxy that routes `eth0` to `tun0` with [dante-server](https://www.inet.no/dante/).

The main advantage is that you get the best recommendation for each selection.

## Usage

[Script](https://github.com/haugene/docker-transmission-openvpn/blob/master/openvpn/nordvpn/updateConfigs.sh) for OpenVpn config download is base on the one developped for [haugene](https://github.com/haugene/docker-transmission-openvpn) 's docker transmission openvpn
https://haugene.github.io/docker-transmission-openvpn/provider-specific/

The container is expecting three informations to select the vpn server:
* [NORDVPN_COUNTRY](https://api.nordvpn.com/v1/servers/countries) define the exit country.
* [NORDVPN_PROTOCOL](https://api.nordvpn.com/v1/technologies) although many protocols are possible, only tcp or udp are available.
* [NORDVPN_CATEGORY](https://api.nordvpn.com/v1/servers/groups) although many categories are possible, p2p seems more adapted.
 
> NOTE: This container works best using the `p2p` technology.
> 
> NOTE: At the moment, this container has no kill switch... meaning that when the VPN connection is down, the connection will be rerouted through your provider. although, on tunnel down event, the socks server is stopped preventing to relay unprotected requests.   

* DNS to uses external DNS, if none given: "1.1.1.1@853#cloudflare-dns.com 1.0.0.1@853#cloudflare-dns.com"
* NORDVPN_USER=email or service user
* NORDVPN_PASS=pass or service pass
* EXIT_WHEN_IP_NOTEXPECTED=(0|1) # stop container when detected network is not as expected (based on /24 networks)

```bash
docker run -it --rm --cap-add NET_ADMIN -p 1080:1080 -e NORDVPN_USER=<email> -e NORDVPN_PASS='<pass>' -e NORDVPN_COUNTRY=Poland
 -e NORDVPN_PROTOCOL=udp -e NORDVPN_CATEGORY=p2p   edgd1er/nordvpn-proxy
```

```yaml
version: '3.8'
services:
  proxy:
    image: edgd1er/nordvpn-proxy:latest
    restart: unless-stopped
    ports:
      - "1080:1080"
    devices:
      - /dev/net/tun
    sysctls:
      - net.ipv4.conf.all.rp_filter=2
    cap_add:
      - SYS_MODULE
      - NET_ADMIN
    environment:
      - TZ=America/Chicago
      - DNS=1.1.1.1@853#cloudflare-dns.com 1.0.0.1@853#cloudflare-dns.com
      - NORDVPN_COUNTRY=germany
      - NORDVPN_PROTOCOL=udp
      - NORDVPN_CATEGORY=p2p
      - NORDVPN_LOGIN=<email> #Not required if using secrets
      - NORDVPN_PASS=<pass> #Not required if using secrets
      - EXIT_WHEN_IP_NOTASEXPECTED=0 # when detected ip is not belonging to remote vpn network
      - LOCAL_NETWORK=192.168.53.0/24
      - TINYPORT=8888 #define tinyport inside the container, optional, 8888 by default,
      - TINYLOGLEVEL=Info #Critical (least verbose), Error, Warning, Notice, Connect (to log connections without Info's noise), Info
      - DANTE_LOGLEVEL= #Optional, error by default, available values: connect disconnect error data
      - DANTE_ERRORLOG=/dev/stdout #Optional, /dev/null by default
      - DEBUG=0 #(0/1) activate debug mode for scripts, dante, nginx, tinproxy
    secrets:
        - NORDVPN_LOGIN
        - NORDVPN_PASS

secrets:
    NORDVPN_LOGIN:
        file: ./nordvpn_login
    NORDVPN_PASS:
        file: ./nordvpn_pass
```

