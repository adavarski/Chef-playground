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

$ cd ~/chef-repo
$ mkdir environments

[vagrant@chef-workstation environments]$ cat production.rb 
name "production"
description "The master production branch"
cookbook_versions({
    "nginx" => "<= 1.1.0",
})

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



