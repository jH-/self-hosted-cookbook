# nginx-proxy-manager

- it's basically a very nice UI for nginx reverse-proxy
- a very nice UI
- making it extremely easy to use nginx
---
- [Github repo](https://github.com/NginxProxyManager/nginx-proxy-manager)
- [Homepage](https://nginxproxymanager.com/setup)

## docker-compose.yml
```yml
services:
  app:
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    environment:
      # Uncomment this if IPv6 is not enabled on your host
      DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    # other database options are supported, see docs
```
Login with:
- Email: admin@example.com
- Password: changeme
