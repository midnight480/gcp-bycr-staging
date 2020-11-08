
# Enviorment

* CentOS7
* Internet Access(Outbound)

# Settings

## firewalld

```
sudo firewall-cmd --permanent --add-service=http --zone=public
sudo firewall-cmd --permanent --add-service=https --zone=public
sudo firewall-cmd --reload
```

## nginx

`sudo yum -y install nginx`

## Let's Encrypt

`sudo yum -y install certbot`

```
sudo certbot certonly --webroot -w /var/www/html -d external.example.com --email aaa@example.com
```

* nginx.conf

```
server {
       listen         80;
       server_name    external.example.com;
       rewrite        ^ https://$http_host$request_uri? permanent;
}

server {
       listen         443 ssl;
       server_name    external.example.com;

       ssl on;
       ssl_certificate         /etc/letsencrypt/live/external.example.com/fullchain.pem;
       ssl_certificate_key     /etc/letsencrypt/live/external.example.com/privkey.pem;

       location / {
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-Proto https;
             proxy_set_header X-Forwarded-Host $host;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

             proxy_pass http://on-prem.example.com/;
             proxy_redirect http:// https://;
       }
}
```

## After Install

```
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```
