# server_deployment_instruction
All steps on how to deploy Django and websocket (node socketio) to ubuntu server \
and config https.

Make sure to have all domain names that being used down below pointed to corresponding server

### DNS notes
In domain registra site make sure to point domain name to custom DNS provided by server (digitalocean, linode,...).

In server provider site, make sure to add new A record for domain.com or customhost.domain.com to point to server ip address.

setup firewall:
```
sudo ufw deny incoming
sudo ufw allow outgoing
sudo ufw allow 8000 8000/tcp
sudo ufw allow 80 80/tcp
sudo ufw allow 443 443/tcp
sudo ufw allow 3000 3000/tcp
sudo ufw enable
```

## For Django server

Install nginx and virtual environment for python3
```
sudo apt install nginx
sudo apt install python3-venv
```
Activate the virtual environment and install gunicorn
```
pip3 install gunicorn
```
Create a gunicorn config file in `conf/` directory
```
mkdir conf
cd conf/
touch gunicorn_config.py
```
Write the following codes in `gunicorn_config.py`
```python3
command = "/path/to/venv/bin/gunicorn"
pythonpath = "/path/to/server"
bind = "160.39.192.225:8000" # change to actual server ip address with port 8000
workers = 3

'''
make sure to add ip address and all domain names for server
to ALLOW_HOSTS in Django settings.py.

Also enable cors for client origins if use frontend frameworks
'''
```
Run gunicorn (server.wsgi means server/wsgi.py in Django project)
```
gunicorn -c conf/gunicorn_config.py server.wsgi
```
To make gunicorn run in background press `Ctrl+z` and run
```
bg
```
To kill the background process
```
pkill gunicorn
```

Create nginx config file with the following code
```
vim /etc/nginx/site-availables/server_config
```
```nginx
server {
    listen 80;
    server_name mydomain.com www.mydomain.com customhost.mydomain.com;

    location / {
        proxy_pass http://160.39.192.225:8000;
    }
}
```
If Django project serves static files, please install `whitenoise` to handle all static files.

https://whitenoise.readthedocs.io/en/latest/

Make the nginx conf file live
```
cd /etc/nginx/site-enabled/
sudo ln -s /etc/nginx/site-available/server_config
ls -l
```
Restart nginx
```
sudo systemctl restart nginx
```

## For Websocket

Install Nodejs and NPM
```
sudo apt update
sudo apt install nodejs npm
```
Move to websocket directory
```
cd websocket/
npm install
```
Create nginx config file for websocket with the following code
```nginx
server {
    listen 80;
    server_name websocket.mydomain.com mywebsocketdomain.com;

    location / {
        proxy_pass http://160.39.192.225:3000;
    }
}
```
Repeat the same process as above, add nginx conf file to site-enabled

http://160.39.192.225:3000 because server running on port 3000 

make sure it's the same as port that server is running on

To run node server in background see: https://stackoverflow.com/questions/4018154/how-do-i-run-a-node-js-app-as-a-background-service

## Enable HTTPS for domain names (https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)
Remove certbot-auto and any certbot OS package
```
sudo apt-get remove certbot
```
Install Certbot
```
sudo snap install --classic certbot
```
Prepare the Certbot command
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
Run certbot to automatically edit nginx configurations and serve it
```
sudo certbot --nginx
```
Certbot will ask to select which domain names you want as https

After it finished, notice that our nginx config files have been 
modified by Certbot.

## CloudFlare
After setting up cloudflare, change SSL settings to Full or Full strict HTTPS and update nginx conf to 
```nginx
server {
    listen 80;
    server_name myhost.domain.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name myhost.domain.com;

    ssl_certificate /etc/letsencrypt/live/myhost.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myhost.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http:..... <- ip address and port
    }
}
```
