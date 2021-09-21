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
    image: cloudflare/cloudflared:latest
    command: proxy-dns
    environment:
      - "TUNNEL_DNS_UPSTREAM=https://1.1.1.1/dns-query,https://1.0.0.1/dns-query,https://9.9.9.9/dns-query,https://149.112.112.9/dns-query"
      # Listen on an unprivileged port
      - "TUNNEL_DNS_PORT=5053"
      # Listen on all interfaces
      - "TUNNEL_DNS_ADDRESS=0.0.0.0"
    networks:
      internal:
        ipv4_address: 172.30.10.2

  pihole:
    container_name: pihole
    restart: unless-stopped
    image: pihole/pihole:latest
    environment:
      - "TZ=Europe/Paris"
      - "WEBPASSWORD=admin"
      # Internal IP of the cloudflared container
      - "DNS1=172.30.9.2#5053"
      # Explicitly disable a second DNS server, otherwise Pi-hole uses Google
      - "DNS2=no"
      # Listen on all interfaces and permit all origins
      # This allows Pihole to work in this setup and when answering across VLANS,
      # but do not expose pi-hole to the internet!
      - "DNSMASQ_LISTENING=all"
    volumes:
      - '/mnt/app-data/pihole/config:/etc/pihole/'
      - '/mnt/app-data/pihole/dnsmasq:/etc/dnsmasq.d/'
    # 1. Join the internal network so Pi-hole can talk to cloudflared
    # 2. Join the public network so it's reachable by systems on our LAN
    networks:
      internal:
        ipv4_address: 172.30.10.3
      priv_lan:
        ipv4_address: 192.168.0.100
    # Starts cloudflard before Pi-hole
    depends_on:
      - cloudflared

networks:
  internal:
    ipam:
      config:
        - subnet: 172.30.9.0/29
  # The priv_lan network is already setup, so it is an 'external' network
  priv_lan:
    external:
      name: priv_lan
```

## Option 2: Attach cloudflared to the LAN

Another option is to skip using the `internal` network and instead directly attach cloudflared to our real network. By doing this, we gain the ability to bypass Pi-hole if desired and still have the benefits of DNS over HTTPS. We also get access to the Prometheus metrics published by cloudflared.

We need to make some changes to the configuration for this setup to work.

### Cloudflare LAN IP

With the internal network removed, we need to bring cloudflared onto the real network priv_lan and assign it the IP address 192.168.0.101

### Permissions

If the goal is to make the cloudflared DNS service available to the LAN, we want it on the standard port 53. The problem is the cloudflare/cloudflared Docker image doesn’t run as root so it won’t have permission to bind to a privileged port (i.e. < 1024). We can fix this with a sysctl option `net.ipv4.ip_unprivileged_port_start=53`

### Prometheus metrics

The Prometheus metrics HTTP server apparently has a default behaviour of randomly generating a port to listen on. This is not helpful, so we can fix that by setting an environment variable `TUNNEL_METRICS=0.0.0.0:49312` to bind to all interfaces on port 49312

### Recap

- Assign an IP on our priv_lan to cloudflared
- Grant cloudflared permission to bind to a privileged port
- Configure cloudflared’s Prometheus metrics (optional)
- Point Pi-hole to the new IP of cloudflared
- Remove the internal network

```yaml
version: "3.6"
services:
  cloudflared:
    container_name: cloudflared
    restart: unless-stopped
    image: cloudflare/cloudflared:latest
    command: proxy-dns
    environment:
      - "TUNNEL_DNS_UPSTREAM=https://1.1.1.1/dns-query,https://1.0.0.1/dns-query,https://9.9.9.9/dns-query,https://149.112.112.9/dns-query"
      - "TUNNEL_METRICS=0.0.0.0:49312"
      - "TUNNEL_DNS_ADDRESS=0.0.0.0"
      - "TUNNEL_DNS_PORT=53"
    sysctls:
      - net.ipv4.ip_unprivileged_port_start=53
    networks:
      priv_lan:
        ipv4_address: 192.168.0.101

  pihole:
    container_name: pihole
    restart: unless-stopped
    image: pihole/pihole:latest
    environment:
      - "TZ=Europe/Paris"
      - "DNS1=192.168.0.100#53"
      - "DNS2=no"
      - "DNSMASQ_LISTENING=all"
      - "WEBPASSWORD=admin"
    volumes:
      - '/mnt/app-data/pihole/config:/etc/pihole/'
      - '/mnt/app-data/pihole/dnsmasq:/etc/dnsmasq.d/'
    networks:
      priv_lan:
        ipv4_address: 192.168.0.100

networks:
  priv_lan:
    external:
      name: priv_lan
```
