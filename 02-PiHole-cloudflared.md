# 02 - PiHole feat Cloudflared

cloudflared provides another type of security with DNS over HTTPS. Traditional DNS is insecure and requests can easily be spied on or modified. DNS over HTTPS prevents this by doing what it sounds like: sending your DNS requests over a secure HTTPS connection. Since few devices support DoH, cloudflared acts as a proxy between traditional DNS requests and DNS over HTTPS.

When setting-up Pi-hole, it needs to be configured with the DNS servers it will use to resolve non-blocked requests. By default this is using Google DNS. We would rather not give more data to Google, and we want to use DoH. So, we’ll configure Pi-hole to direct all requests to our running instance of cloudflared.

## Docker macvlan

- https://docs.docker.com/network/macvlan/

With `macvlan`, Docker can create a new network that generates MAC addresses for containers and lets them have routable IPs on our LAN. If we wanted to, we could have multiple Pi-hole instances running on the same machine, each with its own IP listening on port 53.

In the examples to follow, we’ll say our real network is `192.168.0.0/24` and our router is `192.168.0.1`. We can inform Docker of this topology in a network called `priv_lan` that the host is connected to on interface `eth0`.

To create manually the docker network, use this command :
```bash
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
