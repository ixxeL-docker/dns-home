# 01 - PiHole docker

Using PiHole as a DNS for home is easy. Recommended way is docker compose.

Official doc :
- https://github.com/pi-hole/docker-pi-hole

Your PiHole instance will be accessible @ http://<ip-address>/admin

```yaml
version: '3.6'
networks:
  pihole:
    ipam:
      config:
       - subnet: 172.16.0.0/24
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    ports:
      - 53:53/udp
      - 53:53
      - 443:443/tcp
      - 80:80/tcp
    environment:
      TZ: 'Europe/Paris'
      VIRTUAL_HOST: pihole.example.com
      WEBPASSWORD: 'password'
      DNS1: '1.1.1.1'
      DNS2: '1.0.0.1'
    volumes:
      - ./datapihole:/etc/pihole/
      - ./datadnsmasqd:/etc/dnsmasq.d/
    # Recommended but not required (DHCP needs NET_ADMIN)
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    #cap_add:
    #  - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
    networks:
      pihole:
        ipv4_address: 172.16.0.100
```

URLs to use for black listing :
- https://adaway.org/hosts.txt
- https://v.firebog.net/hosts/AdguardDNS.txt
- https://someonewhocares.org/hosts/zero/hosts

You can add these URLs in the section `Group Management\Adlists` of your PiHole WEB GUI, and then update lists in the `Tools\Update Gravity` menu.
