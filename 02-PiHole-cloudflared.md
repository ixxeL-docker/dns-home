# 02 - PiHole feat Cloudflared

cloudflared provides another type of security with DNS over HTTPS. Traditional DNS is insecure and requests can easily be spied on or modified. DNS over HTTPS prevents this by doing what it sounds like: sending your DNS requests over a secure HTTPS connection. Since few devices support DoH, cloudflared acts as a proxy between traditional DNS requests and DNS over HTTPS.

When setting-up Pi-hole, it needs to be configured with the DNS servers it will use to resolve non-blocked requests. By default this is using Google DNS. We would rather not give more data to Google, and we want to use DoH. So, we’ll configure Pi-hole to direct all requests to our running instance of cloudflared.

## Docker macvlan

- https://docs.docker.com/network/macvlan/

With `macvlan`, Docker can create a new network that generates MAC addresses for containers and lets them have routable IPs on our LAN. If we wanted to, we could have multiple Pi-hole instances running on the same machine, each with its own IP listening on port 53.

In the examples to follow, we’ll say our real network is `192.168.0.0/24` and our router is `192.168.0.1`. We can inform Docker of this topology in a network called `priv_lan` that the host is connected to on interface `eth0`.

To create manually the docker network, use this command :
```shell
docker network create -d macvlan \
                      --subnet=192.168.0.0/24 \
                      --gateway=192.168.0.1 \
                      -o parent=eth0 priv_lan
```
 
 You can also create this in docker-compose file, but the `gateway` parameter is only available for docker-compose version 2.x.
 
 ```yaml
version: "2"
networks:
  priv_lan:
    driver: macvlan
    driver_opts: 
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.0.0/24
        - gateway: 192.168.0.1
```

## Option 1: Hidden cloudflared

**Internal network**

In this setup, we create another Docker network named `internal` that both the cloudflared and Pi-hole containers are connected to. This allows Pi-hole to talk to cloudflared without exposing cloudflared to the rest of the network.

This internal network will be 172.30.10.0/29. The /29 netmask provides 5 usable IP addresses (.1 is the virtual router); plenty for this setup.

cloudflared gets the IP 172.30.9.2 and responds to DNS queries on the unprivileged port 5053. We bind the DNS service to 0.0.0.0 to so it listens on all interfaces.

Pi-hole is assigned the IP 172.30.9.2 on our internal network and gets attached to the real network with the IP 192.168.0.100

Pi-hole is configured to use the internal cloudflared as the exclusive DNS server.

```yaml
version: "3.6"
services:
  cloudflared:
    container_name: cloudflared
    restart: unless-stopped
    image: cloudflare/cloudflared
    command: proxy-dns
    environment:
      - "TUNNEL_DNS_UPSTREAM=https://1.1.1.1/dns-query,https://1.0.0.1/dns-query,https://9.9.9.9/dns-query,https://149.112.112.9/dns-query"
      # Listen on an unprivileged port
      - "TUNNEL_DNS_PORT=5053"
      # Listen on all interfaces
      - "TUNNEL_DNS_ADDRESS=0.0.0.0"
    networks:
      internal:
        ipv4_address: 172.30.9.2

  pihole:
    container_name: pihole
    restart: unless-stopped
    image: pihole/pihole
    environment:
      - "TZ=Europe/Berlin"
      - "WEBPASSWORD=admin"

      # Internal IP of the cloudflared container
      - "DNS1=172.30.9.2#5053"

      # Explicitly disable a second DNS server, otherwise Pi-hole uses Google
      - "DNS2=no"

      # Listen on all interfaces and permit all origins
      # This allows Pihole to work in this setup and when answering across VLANS,
      # but do not expose pi-hole to the internet!
      - "DNSMASQ_LISTENING=all"

    # Persist data and custom configuration to the host's storage
    volumes:
      - '/mnt/app-data/pihole/config:/etc/pihole/'
      - '/mnt/app-data/pihole/dnsmasq:/etc/dnsmasq.d/'

    # 1. Join the internal network so Pi-hole can talk to cloudflared
    # 2. Join the public network so it's reachable by systems on our LAN
    networks:
      internal:
        ipv4_address: 172.30.9.3
      priv_lan:
        ipv4_address: 10.65.2.4

    # Starts cloudflard before Pi-hole
    depends_on:
      - cloudflared

networks:
  # Create the internal network
  internal:
    ipam:
      config:
        - subnet: 172.30.9.0/29

  # The priv_lan network is already setup, so it is an 'external' network
  priv_lan:
    external:
      name: priv_lan
```
