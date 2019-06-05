<div>
	<img  src="https://cdn-images-1.medium.com/max/1600/1*_vzcsGSA5YwmNQOCrulfqg.png" align="top" width="110">
	<img  src="https://github.com/kubernetes/kubernetes/raw/master/logo/logo.png" width="100">
</div>
---

# Ansible script usage

> 4 roles exist `common-master/workers` and `kubernetes-master/workers`, they are all executed from the top playbook `/top_play.yaml` 

## Basic usage
```bash
cd $project_root_directory && ansible-playbook top_play.yaml
```
The scipt is intended for ARM architecture ( Master *RockPi4* and workers *Raspberry3* ) and using the premade base images. When respecting these condition the only variable required to change is `/group_vars/all.yaml` -> **workers** ( mac address of worker Pis ). You can also tweak the range of the dhcp server in `/roles/common-master/defaults/main.yaml` -> **host_number** ( default being 15 which gives you 14 workers ). If you want to add more you need to make new certificates. 

## General informations 
* Base images are not included in the repository
* There is no configuration to be done on your local machine, everything is done by either variables inside the script or ansible config file (also in the repo), no need for ssh known hosts because ansible is configured not to ask for them ( not secured ). 
* To be able to use both inventory file and ansible configuration you **need to be located in the root of the project**
* To access workers on the subnet a SSH proxy is used and public key is forwarded through the master node, SSH keys are in `/ssh_keys`
* This inventory is dynamic and hosts are added by the first tasks in the role `common_master`. If for some reason you don't run this role and still want to do worker related tasks, you can add them by providing the varible `add` with this command `ansible-playbook top_play.yaml --extra-vars="add=true"` (value is not important)
* Downloading of the binaries and package manager updates can be skipped by providing the varible `test` with this command `ansible-playbook top_play.yaml --extra-vars="test=true"` (value is not important), you can provide multiple variables by comma space separating them *e.g : `--extra-vars="test=true, add=true"`*
* Workers get their IP and hostnames from their index in the variables workers, *e.g : the first one will be 192.168.0.2 and his name will be pi.1*. You can change the prefix in `/group_vars/all.yaml` -> **workers_prefix** ( Ansible doesn't want dashes in the names )

## Usefull variables 

* Private network address is changed in `/group_vars/all.yaml` -> **private_net** and netmask in `/roles/common_master/defaults/main.yaml` -> **private_mask**
* Public network interface of the master node is in `/host_vars/master.yaml` -> **public_net**, you also need to change the **master_public_suffix** variable ( Address is deduced from the number of zeros on right part of network address and number of octet you give in suffix, no need for netmask )
* Master interfaces names are changed in `/roles/common_master/defaults/main.yaml` -> **private/public_nic**
* DNS server to forward to can be changed in `/roles/common_master/defaults/main.yaml` -> **forwarded_dns**
* Domain name given by the dhcp server can be changed in `/roles/common_master/defaults/main.yaml` -> **domain_name**

## Making new certificates

Certificates are configured to last a year and to accept addresses from `192.168.0.1` to `192.168.0.15`, so if you need to change private network address of add more workers new certificates are needed : 

* Install `CFSSL` and `CFSSLJSON` on you local machine
```bash
# This scripts make binaries executable, move them to /usr/bin and rename bins so you don't have distro and architecture in the name
# Run with bash
for bin in {'https://pkg.cfssl.org/R1.2/cfssl_linux-arm','https://pkg.cfssl.org/R1.2/cfssljson_linux-arm'}; do 
    wget $bin && chmod u+x ${_##*/} && mv $_ /usr/bin/${_%_*}
done 
```
> Then you should change directory to `/pki` so you don't have to move new certificates to it.  
If you want to change the number/IPs of hosts go into `/make_new_certs/kubernetes-csr.json`, to change cert validity time go into `/make_new_certs/ca-config.json` the rest of the configuration ( organiation name, country, etc. ) is in `/make_new_certs/ca-csr.json`.

* Generate CA key, certificate and request
```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# Then verify with 
openssl x509 -in ca.pem -text -noout
```
* Generate cert and cert key
```bash 
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
# You should get a warning, but it's not a problem

# Then you verify with 
openssl x509 -in kubernetes.pem -text -noout
```

## Base Images

In order to have fewer variables in the script, username and password are the same on both images. They need to correspond to `/groups_vars/all.yaml` -> **ansible_user, ansible_become_pass**

In the worker images a cron job is enabled, it checks every minutes if one of it's IPs is in the range `192.168~`, if not it deletes existing addresses and run `dhclient`. That means no user input is required. The the script is in `/usr/bin/getIP`

### Distro links 

#### Master 
*https://dl.radxa.com/rockpi/images/ubuntu/rockpi4b-ubuntu-bionic-minimal-20190104_2101-gpt.img.gz*
#### Workers
*https://downloads.raspberrypi.org/raspbian_lite_latest*

### Walktrough

#### Both password and username are changed :
```bash
# You need to login as root so no process are owned by rock
# First configure root password
sudo passwd 
# Then CTRL+d and login as root 

# Change username and home directory
usermod -l pi rock
groupmod -n pi rock
usermod -d /home/pi -m pi

# Change user password to value of ansible_become_path (default is 'linuxmasterrace' )
passwd
```
#### Openssh server is needed on master:
```bash
sudo apt update
sudo apt install openssh-server
```
**Warning, on the worker base distro openssh server is already installed but service is not enabled...**
```bash 
sudo systemctl enable ssh
```
#### You need to put your public key in authorized_key 
```bash
ssh-copy-id -i $publickeyfile $hostname 
# For workers you need to directly connect with a cable or run master-common role of ansible scripts and then use master as a proxy
```

#### Cron job on worker 
```bash 
# Copy getIP script in /usr/bin and then 
sudo crontab -e # Prompts to chose editor
# Enter the following in cronjob file
1 * * * * /usr/bin/getIP 
```

#### Minimizing partition size
You can then change partition size to the smalest and ansible script will extend partition and file system **Don't only change file system size** although that would be the most efficient way it would break the script... If you're base image is bigger than 3GB you need to change the variable `/groups_vars/all.yaml` -> **bigest_img_size**