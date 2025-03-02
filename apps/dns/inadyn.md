# inadyn
Inadyn, or In-a-Dyn, is a small and simple Dynamic DNS (DDNS) client with HTTPS & IPv6 support

---
[Github repo](https://github.com/troglobit/inadyn)

## docker-compose.yml
```yml
version: '3'

services:
  inadyn:
    image: ghcr.io/troglobit/inadyn:edge
    container_name: inadyn
    volumes:
      - /volume1/Docker/inadyn/inadyn.conf:/etc/inadyn.conf:ro
    #command: -l debug
    restart: unless-stopped

```

## config example
```conf
# In-A-Dyn v2.0 configuration file format
period          = 600
#allow-ipv6      = true
#verify-address  = false
#user-agent      = Mozilla/5.0

provider domene.shop {
    username       = token
    password       = secret
    hostname       = { my.domain, my.sub.domain }
}

custom otherhost {
    username       = token
    password       = secret
    ddns-server    = api.otherhost.no
    ddns-path      = "/v0/dyndns/update?hostname=%h&myip=%i"
    hostname       = my.sub.domain
	#ddns-response  = "Connection: close"
}
```