# 03 - NGINX Reverse Proxy for PiHole web GUI

By default, Pihole doesn't accept TLS encryption for web access. But you can use a NGINX reverse proxy container in front of your Pihole server to force TLS encryption.

You will need certificate for your Pihole server, certificate here : `server.crt` and private key here : `private.key`, and a `nginx.conf` file to specify the configuration of your reverse proxy.

Just add this section to your docker-compose file :

```yaml
  reverse-proxy:
    container_name: nginx_proxy
    restart: unless-stopped
    image: nginx:latest
    volumes:
      - './nginx/nginx.conf:/etc/nginx/nginx.conf'
      - './nginx/certs:/etc/ssl/private'
    ports:
      - 443:443
    networks:
      internal:
        ipv4_address: 172.30.10.3
    depends_on:
      - pihole
```

You will need an internal network and add one IP to your Pihole service as well :

```yaml
networks:
 priv_lan:
   driver: macvlan
   driver_opts:
     parent: eth0
   ipam:
     config:
       - subnet: 192.168.0.0/24
         gateway: 192.168.0.1
 internal:
   ipam:
     config:
       - subnet: 172.30.10.0/29
```

The final docker-compose file looks like this :

```yaml
version: "2.2"
services:
  cloudflared:
    container_name: cloudflared
    restart: unless-stopped
    image: cloudflare/cloudflared:2021.9.1-amd64
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
      - "DNS1=192.168.0.101#53"
      - "DNS2=no"
      - "DNSMASQ_LISTENING=all"
      - "WEBPASSWORD=admin"
      - "VIRTUAL_HOST=pihole.fredcorp.com"
    volumes:
      - './pihole-data:/etc/pihole/'
      - './pihole-dnsmasq:/etc/dnsmasq.d/'
    networks:
      priv_lan:
        ipv4_address: 192.168.0.100
      internal:
        ipv4_address: 172.30.10.2

  reverse-proxy:
    container_name: nginx_proxy
    restart: unless-stopped
    image: nginx:latest
    volumes:
      - './nginx/nginx.conf:/etc/nginx/nginx.conf'
      - './nginx/certs:/etc/ssl/private'
    ports:
      - 443:443
    networks:
      internal:
        ipv4_address: 172.30.10.3
    depends_on:
      - pihole

networks:
 priv_lan:
   driver: macvlan
   driver_opts:
     parent: eth0
   ipam:
     config:
       - subnet: 192.168.0.0/24
         gateway: 192.168.0.1
 internal:
   ipam:
     config:
       - subnet: 172.30.10.0/29
```

This is very important to specify the `VIRTUAL_HOST` environment variable to your Pihole FQDN otherwise you will encounter JSON format issues on pihole menu.

Here is the `nginx.conf` file :

```
events {

}

http {
  upstream pihole {
    server pihole:80;
  }
  server {
    server_name pihole.fredcorp.com;
    listen 443 ssl;

    ssl_certificate /etc/ssl/private/server.crt;
    ssl_certificate_key /etc/ssl/private/private.key;

    location / {
      proxy_pass http://pihole;
    }
  }
}
```

