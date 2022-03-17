## Preparing your instance

### OS Updates
Before doing anything else, update your operating system by running these two commands:

```
sudo apt update
sudo apt upgrade
```

### Installing Node.js v17.x:

```
curl -fsSL https://deb.nodesource.com/setup_17.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### NGINX
The last piece to install is NGINX. This will be our reverse proxy accepting requests from the internet and forwarding them to n8n. It can easily be extended in case you want to run additional applications on the same instance.

```
sudo apt install nginx
```

Make sure NGINX is running:
The below command should return `Active: active (running)` among other information.
```
sudo systemctl status nginx
```

### Configure NGINX 1st way

The last step is making your n8n instance available to the outside world. To do so, we create what NGINX calls a server block (a configuration block that defines how a server responds to requests):

```
cd /etc/nginx/sites-available/
sudo nano n8n.conf
```

Now insert a copy of the below example configuration and replace
* `<domain name>` with the domain you have set up (**but without HTTP/HTTPS**), e.g. `n8n.mutedjam.com`

```
server {
    server_name <domain name>;
    listen 80;
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

Exit the editor by pressing `Ctrl+X` and then confirm with `Y` when asked whether you want to save the changes made.

Now link the file we have just created to the `sites-enabled` folder and afterwards test your configuration to make sure everything is understood by NGINX:

```
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

You should see an output ending with `nginx: configuration file /etc/nginx/nginx.conf test is successful`. We can now proceed with reload NGINX to apply the newly added configuration:

```
sudo systemctl reload nginx
```

### Configure NGINX 2nd way (on official n8n-pm2 blog)

First, start and enable Nginx by executing the following commands:

```
sudo systemctl start nginx
sudo systemctl enable nginx
```

Next, create a configuration file by executing the command `sudo vi /etc/nginx/conf.d/n8n.conf`. Add the following configuration and save the file.

Copy and paste below code:

```
server {
    server_name subdomain.example.com;
    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Host $host;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

### Install PM2 globally with the following command:

```
sudo npm install pm2 -g
```

### Install n8n globally with the following command:

```
sudo npm install n8n -g
```
Configure environment variable with [official document](https://docs.n8n.io/reference/environment-variables.html)

### Start n8n via PM2

```
pm2 start n8n
```

### Auto-start n8n on machine restart

```
pm2 startup
```

The above command asks you to run another command. Copy and paste the suggested command. The comment would look like this:

```
sudo  "env PATH=$PATH:/user/home/.nvm/versions/node/v17.x/bin pm2 startup <distribution> -u <user> --hp <home-path>
```

PM2 will now automatically restart at boot.

### To update n8n, follow these three steps:

1. Stop the n8n service

```
pm2 stop n8n
```
2. Install the latest version of n8n

```
sudo npm install -g n8n@latest
```
3. Restart the n8n service

```
pm2 restart n8n
```

### Configure environment variables via a config file. 

Execute the command `pm2 init simple` to generate a simple configuration file.

Open the generated file, and replace the existing code with the below code snippet:

```
module.exports = {
    apps : [{
        name   : "n8n",
        env: {
            N8N_BASIC_AUTH_ACTIVE:true,
            N8N_BASIC_AUTH_USER:"USERNAME",
            N8N_BASIC_AUTH_PASSWORD:"PASSWORD",
            N8N_PROTOCOL: "https",
            WEBHOOK_TUNNEL_URL: "https://subdomain.example.com/",
            N8N_HOST: "subdomain.example.com"
        }
    }]
}
```

### SMTP Setup to Invite Users

```
export N8N_SMTP_PORT=587
export N8N_SMTP_HOST=smtp.gmail.com
export N8N_SMTP_USER=<youremail@gmail.com>
export N8N_SMTP_PASS=<yourpassword>
export N8N_SMTP_SENDER=<youremail@gmail.com>
export N8N_EMAIL_MODE=smtp
export N8N_EDITOR_BASE_URL=<subdomain.domain.com>
```

Now, to start n8n, execute the command:

```
pm2 start ecosystem.config.js
```
You can learn more about the configuration files in PM2 on the official [PM2 documentation.](https://pm2.keymetrics.io/docs/usage/application-declaration/)

### Setting up Firewall (UFW)

In this example setup, we will be using [UFW](https://help.ubuntu.com/community/UFW) to configure [iptables](https://help.ubuntu.com/community/IptablesHowTo). Oracle's Ubuntu image does, however, come with iptables-persistent which would need to be removed. To do so, simply run:

```
sudo apt remove iptables-persistent
```

At this stage, reboot your instance to make sure all previous changes have taken effect.

```
sudo reboot now
```

We are now ready to configure UFW. First make sure it knows about all your applications by running the below command. It should return `Nginx Full` and `OpenSSH` among the available applications.

```
sudo ufw app list
```

Now allow both `Nginx Full` and `OpenSSH` to be accessed from the internet:

```
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
```

Lastly, enable `UFW`:

```
sudo ufw enable
```

## Test the webserver
On your local machine, open a browser and navigate to `http://<public IP>` (replace the IP address with the one you copied earlier). You should see the default Nginx landing page "Welcome to NGINX! (...)"
