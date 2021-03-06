# Setting Up Wings on an ARM based Oracle Cloud With Oracle Linux

This guide explains the process of setting up and installing Wings on an ARM based Oracle Cloud machine running Oracle Linux 8. Note that this guide does not explain the panel instalation process. Since the panel requires PHP 8, it is currently reccomended that Ubunutu be used instead  if you would like to install the panel on your Oracle Cloud machine, as PHP 8 will not be availible for RHEL-based ARM distributions until version 8.6.

## Contents

- [Setting Up Wings on an ARM based Oracle Cloud With Oracle Linux](#setting-up-wings-on-an-arm-based-oracle-cloud-with-oracle-linux)
  * [Grub Configuration](#grub-configuration)
  * [Installing Dependencies](#installing-dependencies)
    + [Base Dependencies](#base-dependencies)
    + [Docker](#docker)
    + [SELinux](#selinux)
  * [SELinux Setup](#selinux-setup)
  * [Obtaining SSL Certificates](#obtaining-ssl-certificates)
    + [acme.sh Installation](#acmesh-installation)
    + [Obtaining and Specifying a Cloudflare API Key](#obtaining-and-specifying-a-cloudflare-api-key)
    + [Changing the Default Certificate Provider](#changing-the-default-certificate-provider)
    + [Obtaining the Certificates](#obtaining-the-certificates)
    + [Restarting Wings on Certificate Renewal](#restarting-wings-on-certificate-renewal)
  * [Firewall Setup](#firewall-setup)
    + [FirewallD Setup](#firewalld-setup)
    + [OCI Firewall Setup](#oci-firewall-setup)
  * [Installing Wings](#installing-wings)
    + [Downloading](#downloading)
    + [Configuring](#configuring)
    + [Setting up the Service](#setting-up-the-service)
  * [Allocations](#allocations)

## Grub Configuration

Some changes to the grub configuration are needed for Wings to properly utilize swap, as well as allowing the panel to properly display CPU and RAM usage for the node.

Open the grub config:

```sh
sudo nano /etc/default/grub
```

Add the arguments `swapaccount=1` and `systemd.unified_cgroup_hierarchy=1` to the end of the entry `GRUB_CMDLINE_LINUX`. Ensure each argument is seperated by a space and is within the existing double quotes. Do not delete any existing arguments on the line. Use `Ctrl+X` once done editing, and press `Y` when prompted to save the file.

Run the following to update the grub config:

```sh
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
sudo reboot
```

On reboot, the new grub configuration will be applied.

## Installing Dependencies

### Base Dependencies

To install the base dependencies, run:

```sh
sudo dnf install -y dnf-utils device-mapper-persistent-data lvm2
```

### Docker

To install and enable Docker, run:

```sh
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
```

### SELinux

To install tools for managing SELinux, run:

```sh
sudo dnf install -y policycoreutils selinux-policy selinux-policy-targeted setroubleshoot-server setools setools-console mcstrans
```

## SELinux Setup

To ensure Wings does not encounter errors due to SELinux, run:

```sh
sudo audit2allow -a -M http_port_t
sudo semodule -i http_port_t.pp
```

## Obtaining SSL Certificates

### acme.sh Installation

It is reccomended that you use acme.sh to obtain SSL certificates. It is also reccomended that you run acme.sh as the root user. To switch to the root user, run:

```sh
sudo su -
```

To install acme.sh, run:

```sh
curl https://get.acme.sh | sh
```

### Obtaining and Specifying a Cloudflare API Key

You must obtain an API key from Cloudflare for acme.sh to run properly. Ensure that a DNS record (A or CNAME record) is pointing to your Oracle Cloud machine, and set the cloud to grey (bypassing CloudFlare proxy). Then go to My Profile > API keys and on Global API Key subtab, click on "view", enter your CloudFlare password, and copy the API key to clipboard. To pass acme.sh the API key, run:

```sh
export CF_Key="API_KEY_HERE"
export CF_Email="CLOUDFLARE_ACCOUNT_EMAIL_HERE@example.com"
```

### Changing the Default Certificate Provider

Before continuing, it is reccomended that you change the default certificate provider for acme.sh from ZeroSSL to LetsEncrypt, as ZeroSSL tends to be unreliable. To do so, run:

```sh
/root/.acme.sh"/acme.sh --set-default-ca  --server  letsencrypt
```

### Obtaining the Certificates

To setup the directory for the certificates (being sure to replace `example.com` with your domain or subdomain), run:

```sh
mkdir -p /etc/letsencrypt/live/example.com
```

Finally, to obtain your SSL certificates and setup automatic renewal (being sure to replace `example.com` with your domain or subdomain), run:

```sh
/root/.acme.sh"/acme.sh --issue --dns dns_cf -d "example.com" --key-file /etc/letsencrypt/live/example.com/privkey.pem --fullchain-file /etc/letsencrypt/live/example.com/fullchain.pem
```

### Restarting Wings on Certificate Renewal

It is reccomended that you restart Wings when the cronjob to renew the certificate runs so that Wings picks up the new certificate. To access the crontab file, run:

```sh
EDITOR=nano crontab -e
```

At the end of the acme.sh renewal line, add ` && systemctl restart wings`. Use `Ctrl+X` once done editing, and press `Y` when prompted to save the file.

To switch back to your own user account (being sure to replace yourusername with your username), run:

```sh
su yourusername
```

## Firewall Setup

### FirewallD Setup

To configure FirewallD to work with Wings, run:

```sh
sudo firewall-cmd --add-port 8080/tcp --permanent
sudo firewall-cmd --add-port 2022/tcp --permanent
sudo firewall-cmd --reload
```

### OCI Firewall Setup

From the OCI Console, open the menu using the three horizonal bars in the top left corner of the screen. Select the "Networking" submenu. From within the networking submenu, select "Virtual Cloud Networks". From the list, select the virtual cloud network (VNC) of your machine. From the subnets list, select the subnet of your machine. From the list of security lists, select the security list for your machine's VNC.

Wings requires ports 8080 and 2022 over TCP. Add an ingress rule with source type `CIDR` and a CIDR of `0.0.0.0/0`. Select `TCP` for "IP Protocol" Enter the desired port number for "Destination Port Range" and add the ingress rule.

Note that you must return to this menu in the future to add ports for any servers you deploy on the node, or they will not be accessible publicly. You may disable this firewall by creating an ingress rule  with source type `CIDR`, a CIDR of `0.0.0.0/0`, and an "IP Protocol" of `All Protocols`.

## Installing Wings

### Downloading

To install the Wings executable, run:

```sh
mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings "https://github.com/kas-tle/wings-oci/releases/latest/download/wings_oracle_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
chmod u+x /usr/local/bin/wings
```

### Configuring

Use the panel to create a new node and obtain a Wings configuration file. For the entry `system.data` in the config file, it is reccomended that you use `/var/srv/containers/pterodactyl` to avoid issues with SELinux. To create this directory, run:

```sh
sudo mkdir -p /var/srv/containers/pterodactyl
```

Once you have obtained and copied the config.yml file, run:

```sh
sudo nano /etc/pterodactyl/config.yml
```

Paste the contents of the file, being sure `system.data` uses the modified path. Use `Ctrl+X` once done editing, and press `Y` when prompted to save the file.

### Setting up the Service

Wings must be setup as a service to run in the background and persist on restarts. To create the service file, run:

```sh
sudo nano /etc/systemd/system/wings.service
```

Paste the following contents into the file:

```
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Use `Ctrl+X` once done editing, and press `Y` when prompted to save the file.

To enable and start the service, run:

```sh
sudo systemctl enable --now wings
```

## Allocations

Note that your Oracle Cloud machine will generally not be bound to its public IP address. To check what IP address to use for node allocations, run:

```sh
hostname -I | awk '{print $1}'
```
