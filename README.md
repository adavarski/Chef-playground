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
Once installed, access the Web UI using the URL https://serverip/login. On the login page that appears, provide the admin username and password created above.

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

