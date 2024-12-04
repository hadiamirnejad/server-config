# Server config
# Update Server
`sudo apt update`.
`sudo apt upgrade`

## Install Docker
Run the following command to uninstall all conflicting packages:
```for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done```

1. Set up Docker's `apt` repository.

   ```bash
   # Add Docker's official GPG key:
   sudo apt-get update
   sudo apt-get install ca-certificates curl
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL {{% param "download-url-base" %}}/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc

   # Add the repository to Apt sources:
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] {{% param "download-url-base" %}} \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   ```

   > [!NOTE]
   >
   > If you use an Ubuntu derivative distribution, such as Linux Mint,
   > you may need to use `UBUNTU_CODENAME` instead of `VERSION_CODENAME`.

2. Install the Docker packages.

   {{< tabs >}}
   {{< tab name="Latest" >}}

   To install the latest version, run:

   ```console
   $ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
  
   {{< /tab >}}
   {{< tab name="Specific version" >}}

   To install a specific version of Docker Engine, start by listing the
   available versions in the repository:

   ```console
   # List the available versions:
   $ apt-cache madison docker-ce | awk '{ print $3 }'

   5:27.3.1-1~ubuntu.24.04~noble
   5:27.3.0-1~ubuntu.24.04~noble
   ...
   ```

   Select the desired version and install:

   ```console
   $ VERSION_STRING=5:27.3.1-1~ubuntu.24.04~noble
   $ sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
   ```

   {{< /tab >}}
   {{< /tabs >}}

3. Verify that the installation is successful by running the `hello-world` image:

   ```console
   $ sudo docker run hello-world
   ```

   This command downloads a test image and runs it in a container. When the
   container runs, it prints a confirmation message and exits.

You have now successfully installed and started Docker Engine.

{{< include "root-errors.md" >}}


# Install Portainer
First, create the volume that Portainer Server will use to store its database:
`docker volume create portainer_data`

Then, download and install the Portainer Server container:
`docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.21.4`
> [!NOTE]
> By default, Portainer generates and uses a self-signed SSL certificate to secure port 9443. Alternatively you can provide your own SSL certificate during installation or via the Portainer UI after installation is complete.

> [!NOTE]
> If you require HTTP port 9000 open for legacy reasons, add the following to your `docker run` command:
> `-p 9000:9000`

Portainer Server has now been installed. You can check to see whether the Portainer Server container has started by running `docker ps`

# Install Caddy
Stable releases:

`sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl`
`curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg`
`curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list`
`sudo apt update`
`sudo apt install caddy`

# Domain Setting
> [!NOTE]
> For each domain or subdomain, an A record must be added to the DNS server.
> ```
> Name/Host: utrend.ir
> Type: A
> Value: Your server's IP address
> TTL: 3600 (or as needed)
> ```
> ![image](https://github.com/user-attachments/assets/908c78ce-dfdb-4a7b-8b78-a7998f840e8b)

1. Go to `/etc/caddy/Caddyfile`
2. Edit `Caddyfile`:
   For example, to set up DNS for utrend.ir:

   ````
   {
    email hadiamirnejad@gmail.com  # ایمیل خود را وارد کنید
    http_port 80
    https_port 443
   }
   
   utrend.ir {
      root * /var/www/html/utrend-front
      
      # این خط مهم است - همه مسیرها را به index.html هدایت می‌کند
      try_files {path} /index.html
  
      # تنظیمات قبلی
      encode gzip zstd
      
      header {
          Strict-Transport-Security "max-age=31536000; includeSubDomains"
          X-XSS-Protection "1; mode=block"
          X-Frame-Options "SAMEORIGIN"
          X-Content-Type-Options "nosniff"
          -Server
      }
      
      log {
          output file /var/log/caddy/utrend.ir.log {
              roll_size 10MB
              roll_keep 10
          }
      }
  
      file_server
  
      @www host www.utrend.ir
      redir @www https://utrend.ir{uri} permanent
   }
   import conf.d/*
   ````
3. Create a folder
`mkdir -p /etc/caddy/conf.d`

4. Create a file
   `nano /etc/caddy/conf.d/portainer.utrend.ir`
   The add:

   ```
   portainer.utrend.ir {
    reverse_proxy 127.0.0.1:9000 {
        header_up X-Forwarded-Proto {scheme}
    }
    
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-XSS-Protection "1; mode=block"
        X-Frame-Options "SAMEORIGIN"
        X-Content-Type-Options "nosniff"
        # اضافه کردن دسترسی WebSocket
        Access-Control-Allow-Origin "*"
        -Server
    }

    # اضافه کردن پشتیبانی از WebSocket
    @websocket {
        header Connection *Upgrade*
        header Upgrade websocket
    }
    reverse_proxy @websocket 127.0.0.1:9000
   }
   ```
# Install Postgres with Portainer

