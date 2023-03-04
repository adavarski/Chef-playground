# Chef-playground

### Install VirtualBox & Vagrant Server (Ubuntu)

```
$ sudo apt install virtualbox virtualbox-ext-pack -y
$ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
$ wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
$ sudo apt update
$ sudo apt install vagrant
$ sudo reboot
```

### Setup Virtual environment

For this guide, you will need 3 servers as below.
```
TASK	IP Address	HOSTNAME
--------------------------------------------------------------------------------
Chef-Server(Rocky Linux 8|AlmaLinux) : 192.168.56.11: chef-server.example.com
Chef Workstation : 	192.168.56.12 :	chef-workstation.example.com
Chef Client :	192.168.56.13 :	chef-client.example.com
--------------------------------------------------------------------------------

```

```
$ sudo su -
# mkdir CHEF-LAB && cd CHEF-LAB
# cat Vagrantfile 
IMAGE_NAME = "almalinux/8"
N = 3

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 4096
        v.cpus = 2
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.56.#{i + 10}"
            node.vm.hostname = "node-#{i}"
        end
    end
end

# vagrant up
```

Server hostname can be set as below.
```
sudo hostnamectl set-hostname chef-server.example.com --static
sudo hostnamectl set-hostname chef-workstation.example.com --static
sudo hostnamectl set-hostname chef-client.example.com --static
```
On all the 3 servers
```
$ sudo vi /etc/hosts
192.168.56.11 chef-server.example.com chefserver
```
### Configure NTP on Rocky Linux 8/ AlmaLinux 8 

Ref: https://techviewleo.com/configure-chrony-ntp-server-on-rocky-almalinux/

### Install Chef Infra Server on Rocky Linux 8/ AlmaLinux 8
```
$ vagrant ssh node-1
$ sudo yum -y install wget
$ VER="14.12.21"
$ wget https://packages.chef.io/files/stable/chef-server/${VER}/el/8/chef-server-core-${VER}-1.el7.x86_64.rpm
$ sudo dnf localinstall chef-server-core-${VER}-1.el7.x86_64.rpm
$ sudo chef-server-ctl reconfigure
$ sudo chef-server-ctl status
$ sudo firewall-cmd --permanent --add-service={http,https}
$ sudo firewall-cmd --reload
$ USERNAME="chefadmin"
$ FIRST_NAME="Chef"
$ LAST_NAME="Administrator"
$ EMAIL="chefadmin@techviewleo.com"
$ PASSWORD="Passw0rd"
$ KEY_PATH="/root/chefadmin.pem"
$ sudo chef-server-ctl user-create ${USERNAME} ${FIRST_NAME} ${LAST_NAME} ${EMAIL} ${PASSWORD} -f ${KEY_PATH}
$ sudo chef-server-ctl user-list
$ sudo chef-server-ctl org-create techviewleo 'Techviewleo, Inc.' \
--association_user chefadmin \
--filename /root/techviewleo-validator.pem
$ sudo chef-server-ctl org-list
$ sudo find /root -name "*.pem"
```

###  (Optional) Install Chef Manage

```
$ sudo chef-server-ctl install chef-manage 
$ sudo chef-server-ctl reconfigure 
$ sudo chef-manage-ctl reconfigure
```
Note: Chef Manage UI via VNC
```
$ sudo apt install tightvncserver
$ sudo apt install xfce4 xfce4-goodies
$ sudo mv ~/.vnc/xstartup ~/.vnc/xstartup.bak
root@devops:~/CHEF-LAB# cat ~/.vnc/xstartup
#!/bin/bash
xrdb $HOME/.Xresources
startxfce4 &
$ sudo chmod +x ~/.vnc/xstartup
$ vncserver 
```
Once installed, access the Web UI using the URL https://192.168.56.11/login. On the login page that appears, provide the admin username and password created above.

### Install and Configure the Chef Workstation.

```
$ VER="22.1.778"
$ wget https://packages.chef.io/files/stable/chef-workstation/${VER}/el/8/chef-workstation-${VER}-1.el8.x86_64.rpm
$ sudo yum localinstall chef-workstation-${VER}-1.el8.x86_64.rpm
$ chef --version
$ knife --version
$ chef generate repo chef-repo
$ mkdir ~/chef-repo/.chef
$ cd chef-repo
$ ssh-keygen -b 4096
$ ssh-copy-id root@192.168.56.11
$ scp root@192.168.56.11:/root/*.pem ~/chef-repo/.chef/
$ ls ~/chef-repo/.chef

```

### Configure Knife and Bootstrap a Client Node

Bootstrapping basically installs the Chef Infra Client on the client system so it can be able to communicate with the Chef server. Basically, there are two ways to bootstrap a client node:

Using knife bootstrap from the workstation
Unattended install to bootstrap from the node without SSH or WinRM connectivity required
For this guide, we will use the Knife bootstrap method. Begin by creating a config.rb file on your workstation under ~/chef-repo/.chef

On the Workstation node:
```
$ vi ~/chef-repo/.chef/config.rb
In the file, add the below lines.

current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                'chefadmin'
client_key               "chefadmin.pem"
validation_client_name   'techviewleo-validator'
validation_key           "techviewleo-validator.pem"
chef_server_url          'https://chef-server.example.com/organizations/techviewleo'
cache_type               'BasicFile'
cache_options( :path => "#{ENV['HOME']}/.chef/checksums" )
cookbook_path            ["#{current_dir}/../cookbooks"]

$ cd ~/chef-repo
$ knife ssl fetch
$ knife client list
$ cd ~/chef-repo/.chef
$ knife bootstrap 192.168.56.13 -x root -P vagrant --node-name chef-client
$ knife node list
$ knife node show chef-client
```

### A basic NGINX cookbook for CHEF


```
$ vagrant ssh node-2
$ cd ~/chef-repo
$ mkdir cookbooks
$ cd cookbooks
$ chef generate cookbook nginx

### Edit the file default.rb in in nginx/recipes/default.rb

$ vi nginx/recipes/default.rb
package 'git'
package 'tree'

package 'nginx' do
  action :install
end


service 'nginx' do
  action [ :enable, :start ]
end


cookbook_file "/var/www/html/index.html" do
  source "index.html"
  mode "0644"
end
template "/etc/nginx/nginx.conf" do   
  source "nginx.conf.erb"
  notifies :reload, "service[nginx]"
end

$ chef generate file index.html

### Edit the file index.html in files/default/index.html

$ vi files/default/index.html 
<html>
  <head>
    <title>Hello there</title>
  </head>
  <body>
    <h1>This is a test</h1>
    <p>Please work!</p>
  </body>
</html>

### Create a templates named nginx.conf:

$ chef generate template nginx.conf

Edit the file nginx.conf.erb in templates/nginx.conf.erb and insert your custom configuration of nginx.

$ vi templates/nginx.conf.erb
user www-data;
worker_processes auto;
pid /run/nginx.pid;
events {
	worker_connections 768;
	# multi_accept on;
}
http {
	##
	# Basic Settings
	##
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;
	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	##
	# SSL Settings
	##
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;
	##
	# Logging Settings
	##
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	##
	# Gzip Settings
	##
	gzip on;
	gzip_disable "msie6";
	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
	##
	# Virtual Host Configs
	##
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}
#mail {
#	# See sample authentication script at:
#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
# 
#	# auth_http localhost/auth.php;
#	# pop3_capabilities "TOP" "USER";
#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
# 
#	server {
#		listen     localhost:110;
#		protocol   pop3;
#		proxy      on;
#	}
# 
#	server {
#		listen     localhost:143;
#		protocol   imap;
#		proxy      on;
#	}
#}

### Upload cookbook and apply for chef-client
$ knife cookbook upload nginx
$ export EDITOR=vi
$ knife node edit chef-client
{
  "name": "chef-client",
  "chef_environment": "_default",
  "normal": {
    "tags": [

    ]
  },
  "policy_name": null,
  "policy_group": null,
  "run_list": [
    "recipe[nginx]"

]

}

### 
$ vagrant ssh node-3
$ sudo chef-client
$ curl localhost
$ sudo yum remove nginx (to move to environments & roles example: production ready)


```

### Use Roles and Environments in Chef to Control Server Configurations (Production-ready configuration)


```
$ vagrant ssh node-2

$ cd ~/chef-repo
$ mkdir roles && cd roles
[vagrant@chef-workstation roles]$ cat web_server.rb 
name "web_server"
description "A role to configure our front-line web servers"
run_list "recipe[nginx]"

$ knife role create web_server
{
  "name": "web_server",
  "description": "",
  "json_class": "Chef::Role",
  "default_attributes": {

  },
  "override_attributes": {

  },
  "chef_type": "role",
  "run_list": [
    "recipe[nginx]"

  ],
  "env_run_lists": {

  }
}
[vagrant@chef-workstation roles]$ knife role create web_server
Created role[web_server]

[vagrant@chef-workstation chef-repo]$ knife role list
web_server

$ cd ~/chef-repo
$ mkdir environments

[vagrant@chef-workstation environments]$ cat production.rb 
name "production"
description "The master production branch"
cookbook_versions({
    "nginx" => "<= 1.1.0",
})

[vagrant@chef-workstation environments]$ knife environment create production
{
  "name": "production",
  "description": "",
  "cookbook_versions": {
    "nginx": "<= 1.1.0"
  },
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "default_attributes": {

  },
  "override_attributes": {

  }
}
[vagrant@chef-workstation environments]$ knife environment create production
Created production

[vagrant@chef-workstation chef-repo]$ knife environment list 
_default
production

$ knife node edit chef-client

{
  "name": "chef-client",
  "chef_environment": "production",
  "normal": {
    "tags": [

    ]
  },
  "policy_name": null,
  "policy_group": null,
  "run_list": [
  "role[web_server]"
]

}

[vagrant@chef-workstation environments]$ knife search "role:web_server AND chef_environment:production" -a name
1 items found

chef-client:
  name: chef-client

[vagrant@chef-workstation environments]$ 

$ vagrant ssh node-3

[vagrant@chef-client ~]$ sudo chef-client 
Chef Infra Client, version 17.10.3
Patents: https://www.chef.io/patents
Infra Phase starting
Resolving cookbooks for run list: ["nginx"]
Synchronizing cookbooks:
  - nginx (0.1.0)
Installing cookbook gem dependencies:
Compiling cookbooks...
Loading Chef InSpec profile files:
Loading Chef InSpec input files:
Loading Chef InSpec waiver files:
Converging 6 resources
Recipe: nginx::default
  * dnf_package[git] action install (up to date)
  * dnf_package[tree] action install (up to date)
  * dnf_package[nginx] action install
    - install version 1:1.14.1-9.module_el8.3.0+2165+af250afe.alma.x86_64 of package nginx
  * service[nginx] action enable
    - enable service service[nginx]
  * service[nginx] action start
    - start service service[nginx]
  * cookbook_file[/usr/share/nginx/html/index.html] action create
    - update content in file /usr/share/nginx/html/index.html from 4b1b70 to e03ac2
    --- /usr/share/nginx/html/index.html	2021-04-19 10:05:11.000000000 +0000
    +++ /usr/share/nginx/html/.chef-index20230304-12776-fgvmr2.html	2023-03-04 13:53:37.968530684 +0000
    @@ -1,118 +1,10 @@
    -<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">
    -
    -<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
    -    <head>
    -        <title>Test Page for the Nginx HTTP Server on AlmaLinux</title>
    -        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    -        <style type="text/css">
    -            /*<![CDATA[*/
    -            body {
    -                background-color: #FAF5F5;
    -                color: #000;
    -                font-size: 0.9em;
    -                font-family: sans-serif,helvetica;
    -                margin: 0;
    -                padding: 0;
    -            }
    -            :link {
    -                color: #0B2335;
    -            }
    -            :visited {
    -                color: #0B2335;
    -            }
    -            a:hover {
    -                color: #0069DA;
    -            }
    -            h1 {
    -                text-align: center;
    -                margin: 0;
    -                padding: 0.6em 2em 0.4em;
    -                background-color: #0B2335;
    -                color: #fff;
    -                font-weight: normal;
    -                font-size: 1.75em;
    -                border-bottom: 2px solid #000;
    -            }
    -            h1 strong {
    -                font-weight: bold;
    -                font-size: 1.5em;
    -            }
    -            h2 {
    -                text-align: center;
    -                background-color: #0B2335;
    -                font-size: 1.1em;
    -                font-weight: bold;
    -                color: #fff;
    -                margin: 0;
    -                padding: 0.5em;
    -                border-bottom: 2px solid #000;
    -            }
    -            hr {
    -                display: none;
    -            }
    -            .content {
    -                padding: 1em 5em;
    -            }
    -            .alert {
    -                border: 2px solid #000;
    -            }
    -
    -            img {
    -                border: 2px solid #FAF5F5;
    -                padding: 2px;
    -                margin: 2px;
    -            }
    -            a:hover img {
    -                border: 2px solid #294172;
    -            }
    -            .logos {
    -                margin: 1em;
    -                text-align: center;
    -            }
    -            /*]]>*/
    -        </style>
    -    </head>
    -
    -    <body>
    -        <h1>Welcome to <strong>nginx</strong> on AlmaLinux!</h1>
    -
    -        <div class="content">
    -            <p>This page is used to test the proper operation of the
    -            <strong>nginx</strong> HTTP server after it has been
    -            installed. If you can read this page, it means that the
    -            web server installed at this site is working
    -            properly.</p>
    -
    -            <div class="alert">
    -                <h2>Website Administrator</h2>
    -                <div class="content">
    -                    <p>This is the default <tt>index.html</tt> page that
    -                    is distributed with <strong>nginx</strong> on
    -                    AlmaLinux.  It is located in
    -                    <tt>/usr/share/nginx/html</tt>.</p>
    -
    -                    <p>You should now put your content in a location of
    -                    your choice and edit the <tt>root</tt> configuration
    -                    directive in the <strong>nginx</strong>
    -                    configuration file
    -                    <tt>/etc/nginx/nginx.conf</tt>.</p>
    -
    -                    <p>For information on AlmaLinux, please visit the <a href="http://www.almalinux.org/">AlmaLinux website</a>.</p>
    -
    -                </div>
    -            </div>
    -
    -            <div class="logos">
    -                <a href="http://nginx.net/"><img
    -                    src="nginx-logo.png"
    -                    alt="[ Powered by nginx ]"
    -                    width="121" height="32" /></a>
    -                <a href="http://www.almalinux.org/"><img
    -                    src="poweredby.png"
    -                    alt="[ Powered by AlmaLinux ]"
    -                    width="124" height="32" /></a>
    -            </div>
    -        </div>
    -    </body>
    +<html>
    +  <head>
    +    <title>Hello there</title>
    +  </head>
    +  <body>
    +    <h1>This is a test</h1>
    +    <p>Please work!</p>
    +  </body>
     </html>
    - restore selinux security context
  * template[/etc/nginx/nginx.conf] action create
    - update content in file /etc/nginx/nginx.conf from 1ada00 to 7ba00b
    --- /etc/nginx/nginx.conf	2021-04-19 10:13:15.000000000 +0000
    +++ /etc/nginx/.chef-nginx20230304-12776-9azppz.conf	2023-03-04 13:53:38.089533088 +0000
    @@ -1,91 +1,69 @@
    -# For more information on configuration, see:
    -#   * Official English Documentation: http://nginx.org/en/docs/
    -#   * Official Russian Documentation: http://nginx.org/ru/docs/
    -
    -user nginx;
    +user www-data;
     worker_processes auto;
    -error_log /var/log/nginx/error.log;
     pid /run/nginx.pid;
    -
    -# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
    -include /usr/share/nginx/modules/*.conf;
    -
     events {
    -    worker_connections 1024;
    +	worker_connections 768;
    +	# multi_accept on;
     }
    -
     http {
    -    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    -                      '$status $body_bytes_sent "$http_referer" '
    -                      '"$http_user_agent" "$http_x_forwarded_for"';
    -
    -    access_log  /var/log/nginx/access.log  main;
    -
    -    sendfile            on;
    -    tcp_nopush          on;
    -    tcp_nodelay         on;
    -    keepalive_timeout   65;
    -    types_hash_max_size 2048;
    -
    -    include             /etc/nginx/mime.types;
    -    default_type        application/octet-stream;
    -
    -    # Load modular configuration files from the /etc/nginx/conf.d directory.
    -    # See http://nginx.org/en/docs/ngx_core_module.html#include
    -    # for more information.
    -    include /etc/nginx/conf.d/*.conf;
    -
    -    server {
    -        listen       80 default_server;
    -        listen       [::]:80 default_server;
    -        server_name  _;
    -        root         /usr/share/nginx/html;
    -
    -        # Load configuration files for the default server block.
    -        include /etc/nginx/default.d/*.conf;
    -
    -        location / {
    -        }
    -
    -        error_page 404 /404.html;
    -            location = /40x.html {
    -        }
    -
    -        error_page 500 502 503 504 /50x.html;
    -            location = /50x.html {
    -        }
    -    }
    -
    -# Settings for a TLS enabled server.
    -#
    -#    server {
    -#        listen       443 ssl http2 default_server;
    -#        listen       [::]:443 ssl http2 default_server;
    -#        server_name  _;
    -#        root         /usr/share/nginx/html;
    -#
    -#        ssl_certificate "/etc/pki/nginx/server.crt";
    -#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
    -#        ssl_session_cache shared:SSL:1m;
    -#        ssl_session_timeout  10m;
    -#        ssl_ciphers PROFILE=SYSTEM;
    -#        ssl_prefer_server_ciphers on;
    -#
    -#        # Load configuration files for the default server block.
    -#        include /etc/nginx/default.d/*.conf;
    -#
    -#        location / {
    -#        }
    -#
    -#        error_page 404 /404.html;
    -#            location = /40x.html {
    -#        }
    -#
    -#        error_page 500 502 503 504 /50x.html;
    -#            location = /50x.html {
    -#        }
    -#    }
    -
    +	##
    +	# Basic Settings
    +	##
    +	sendfile on;
    +	tcp_nopush on;
    +	tcp_nodelay on;
    +	keepalive_timeout 65;
    +	types_hash_max_size 2048;
    +	# server_tokens off;
    +	# server_names_hash_bucket_size 64;
    +	# server_name_in_redirect off;
    +	include /etc/nginx/mime.types;
    +	default_type application/octet-stream;
    +	##
    +	# SSL Settings
    +	##
    +	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    +	ssl_prefer_server_ciphers on;
    +	##
    +	# Logging Settings
    +	##
    +	access_log /var/log/nginx/access.log;
    +	error_log /var/log/nginx/error.log;
    +	##
    +	# Gzip Settings
    +	##
    +	gzip on;
    +	gzip_disable "msie6";
    +	# gzip_vary on;
    +	# gzip_proxied any;
    +	# gzip_comp_level 6;
    +	# gzip_buffers 16 8k;
    +	# gzip_http_version 1.1;
    +	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    +	##
    +	# Virtual Host Configs
    +	##
    +	include /etc/nginx/conf.d/*.conf;
    +	include /etc/nginx/sites-enabled/*;
     }
    -
    +#mail {
    +#	# See sample authentication script at:
    +#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
    +# 
    +#	# auth_http localhost/auth.php;
    +#	# pop3_capabilities "TOP" "USER";
    +#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
    +# 
    +#	server {
    +#		listen     localhost:110;
    +#		protocol   pop3;
    +#		proxy      on;
    +#	}
    +# 
    +#	server {
    +#		listen     localhost:143;
    +#		protocol   imap;
    +#		proxy      on;
    +#	}
    +#}
    - restore selinux security context
  * service[nginx] action reload
    - reload service service[nginx]

Running handlers:
Running handlers complete
Infra Phase complete, 6/8 resources updated in 15 seconds
[vagrant@chef-client ~]$ curl localhost
<html>
  <head>
    <title>Hello there</title>
  </head>
  <body>
    <h1>This is a test</h1>
    <p>Please work!</p>
  </body>
</html>
[vagrant@chef-client ~]$ 
```
