# Setup standalone server in Xcloud

MacOS and iOS engines are built by standalone servers running Apple hardware. Our mac cloud provider is [Xcloud](https://xcloud.me/).

Here is how to setup a standalone server in Xcloud.

### Open the Xcloud web console

1. Login to the [Xcloud customer portal](https://my.flow.ch) (a personal account can be added by another Xcloud user)
2. Go to Cloud Server -> Control Panel
3. Choose which server to open and click "Action" -> "Open Console"

### Change password

First off, change the Xcloud user's password you got from Xcloud. Open up a terminal, logged in as user "Xcloud" and type:

`passwd`

### Initial locale preferences

1. Open System Preferences -> Keyboard -> Input Sources
   1. Remove "Swiss German"
1. Open System Preferences -> Language & Region
   1. Remove "Deutsch/German" in Preferred Languages
   1. Set region to "Sweden"

### Set timezone to UTC

The Extender build server in Amazon Web Services run in UTC timezone, so let's use the same timezone here:

`sudo ln -sfn /usr/share/zoneinfo/UTC /etc/localtime`

Reboot the machine to get the new timezone.

### Enable SSH

1. Open System Preferences -> Sharing
   1. Turn on "Remote Login" service
      1. Allow access for: Xcloud
      1. Remove access for: Administrator
1. Open System Preferences -> Security & Privacy -> Firewall
   1. Make sure the firewall is enabled
   1. Click the lock (bottom left) to make changes
   1. Click "Firewall Options"
      1. Allow incoming connections for "/usr/libexec/sshd-keygen-wrapper" (tip: drag libexec folder into "Open dialog" from Finder to list that directory)

From now on, you can use SSH to execute commands.

### Restrict SSH to King office/VPN IP

Open the SSH configuration with an editor:

`sudo nano /etc/ssh/sshd_config` 

Add the following at the bottom of the file:

`AllowUsers xcloud@82.99.54.98`

This tells the SSH server to only allow SSH logins for the user "xcloud" from IP 82.99.54.98 (King external IP)

### Add a hostname for the standalone mac server

1. Surf to the [King SSO portal](https://sso.king.com)
1. Click on "Amazon Web Services - Defold" to open the AWS console
1. Go to Route53 -> Hosted zones -> defold.com
1. Click "Create Record Set" and enter values
    * Name: The name, e.g. "build-darwin-stage"
    * Type: A - IPv4 address
    * Alias: No
    * Value: Xcloud IP address to machine
1. Click "Create"

Now try to ping the host from a local computer:

`ping build-darwin-stage.defold.com`

If ping doesn't find it, wait for the name servers to catch up and try again.

### Install Homebrew

Homebrew requires Xcode CLI tools. The Xcode CLI tools installer will open a GUI dialog with license agreement, so open the Xcloud web console and run the following command from the terminal there:

`xcode-select --install`

Install Homebrew:

`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`   

### Install Java 8

Install Java 8 with HomeBrew:

`brew tap caskroom/versions`  
`brew cask install java8`

### Install utilities needed by setup/service scripts

`brew install tree wget`

### Add defold.com SSL certificate

Create a directory where the SSL certificates will be stored:

`mkdir /usr/local/etc/ssl`

Secure copy the certificate files (located in Defold team drive in Google Drive) to that directory from your local computer:

`scp aws.wildcard.defold.com.* xcloud@build-darwin-stage.defold.com:/usr/local/etc/ssl/`

### Install nginx

Install nginx to enable HTTPS with SSL certificates for the web application:

`brew install nginx`

Start nginx and make sure it starts up at boot:

`sudo brew services start nginx`  

Go to Xcloud web console and there will hopefully be a firewall popup: allow nginx to accept incoming connections.

### Configure nginx

Note: the nginx configuration should be backed up in `/usr/local/etc/nginx/nginx.conf.default`.

Edit the nginx configuration:

`nano /usr/local/etc/nginx/nginx.conf`

1. Make nginx listen to HTTPS port (443) instead of HTTP (port 80) and add SSL certificates.

   Replace this:
   ```
           listen       8080;
           server_name  localhost;
   ```
   With this:
   ```
           listen      443 ssl;
           
           ssl_certificate /usr/local/etc/ssl/aws.wildcard.defold.com.pem;
           ssl_certificate_key /usr/local/etc/ssl/aws.wildcard.defold.com.key;
           
           server_name  build-darwin-stage.defold.com;
   ```
1. Pass all requests to the web application running on port 8080. Raise max file upload size to 500MB.

   Replace this:
   ```
           root   html;
           index  index.html index.htm;
   ```
   With this:
   ```
           proxy_pass http://localhost:8080/;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_set_header X-Forwarded-Port $server_port;
           client_max_body_size 500M;
           proxy_request_buffering off;
           proxy_buffering off;
   ```
1. Comment out the error page.
   
   Replace this:
   ```
           error_page   500 502 503 504  /50x.html;
           location = /50x.html {
               root   html;
           }
   ```
   With this:
   ```
           #error_page   500 502 503 504  /50x.html;
           #location = /50x.html {
           #    root   html;
           #}
   ```

Restart nginx:

`brew services restart nginx`

### Configure firewall

Create a new firewall anchor:

`sudo nano /etc/pf.anchors/com.defold`

Add the following rules to the anchor:

```
# Restrict SSH (port 22) access to King office/VPN IP
block return in proto tcp from any to any port 22
pass in inet proto tcp from 82.99.54.98 to any port 22 no state

# Block all HTTP (port 8080) access from outside (network interface en0)
block return on en0 inet proto tcp from any to any port 8080

# Allow HTTPS (port 443) access
pass in inet proto tcp from any to any port 443
```

Add the defold anchor to the default PF configuration file `/etc/pf.conf`. This allows the anchor and the rules to be active whenever you activate the macOS firewall without interfering with any application firewall rule defined through the GUI.

Open the default PF configuration file:

`sudo nano /etc/pf.conf`

Add the anchor to the bottom of the file:

```
anchor "defold"
load anchor "defold" from "/etc/pf.anchors/com.defold"
```

Enable the firewall and activate the defold rules:

`sudo pfctl -ef /etc/pf.conf`

### Extender service

Create the Extender home:

`sudo mkdir /usr/local/extender`
`sudo chown xcloud:admin /usr/local/extender`

Create the Extender environment properties file, which is used by the service script. Open the file:
`nano /usr/local/extender/env.properties`

Add the following (for production):

`profile=standalone-production`

Add the following (for stage):

`profile=standalone-stage`

# Operations

### What needs to run?

The following services need to run:

* NGINX load balancer & proxy
* Extender
* PF firewall

Start nginx:

`brew services start nginx`

Start extender service:

`extender start`

Enable PF firewall:

`sudo pfctl -ef /etc/pf.conf`

### Where are the logs?

* NGINX access and error logs: /usr/local/var/log/nginx/
* Extender service logs: /usr/local/extender/logs/

### Xcloud services

* [Xcloud portal with web console](https://my.flow.ch/portal/cloudserver)
* [Status page with incidents](https://status.flow.ch/)