# OpenVPN server setup with docker

This manual is based on awesome Kyle Manna's [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) repo and article [How To Run OpenVPN in a Docker Container on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-run-openvpn-in-a-docker-container-on-ubuntu-14-04), but updated for Ubuntu 16.04.

* You need your own VPS with Ubuntu 16.04 with ssh access
* Check firewall settings of your VPS provider and open UDP 1194 port
* In all examples replace: 
  * `user_name` with your VPS user name (`ubuntu`, by default on AWS)
  * `server_ip` with your VPS external IP address
  
## About OpenVPN
  
OpenVPN is an open-source software application that implements virtual private network (VPN) techniques to create secure point-to-point or site-to-site connections in routed or bridged configurations and remote access facilities. It uses a custom security protocol that utilizes SSL/TLS for key exchange. It is capable of traversing network address translators (NATs) and firewalls. It was written by James Yonan and is published under the GNU General Public License (GPL).

OpenVPN has several ways to authenticate peers with each other. OpenVPN offers pre-shared keys, certificate-based, and username/password-based authentication. Preshared secret key is the easiest, and certificate-based is the most robust and feature-rich.

## Install docker

Update your system packages database:
```
sudo apt-get update
```
Add GPG key of Docker's official repository:
```
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
```
Add Docker repository to apt:
```
sudo apt-add-repository 'deb https://apt.dockerproject.org/repo ubuntu-xenial main'
```
Install Docker:
```
sudo apt-get update & sudo apt-get install -y docker-engine
```
After this command Docker must be installed, Docker daemon running and the process should start at system startup. Check if the process is running:
```
sudo systemctl status docker
```
To run Docker without `sudo` command your user need to be in `docker` group:
```
sudo usermod -aG docker $(whoami)
```
To check if Docker installed propely:
```
docker run hello-world
```
Output:
```
Hello from Docker.
This message shows that your installation appears to be working correctly.
...
```

## Set Up the EasyRSA PKI Certificate Store

We will need Docker volume container to store OpenVPN configuration and certificates. Just call it "ovpn-data" (you can choose any name you like). More on Docker volumes in [documentation](https://docs.docker.com/storage/volumes/).

Create an empty Docker volume container using busybox as a minimal Docker image:
```
docker run --name ovpn-data -v /etc/openvpn busybox
```
To initialize configuration files you need domain name for your VPN server. Alternatively, it's possible to use just the IP address of the server, but this is not recommended. Replace `vpn.example.com` with your fully-qualified domain name:
```
docker run --volumes-from ovpn-data --rm kylemanna/openvpn ovpn_genconfig -u udp://vpn.example.com:1194
```
Generate the EasyRSA PKI certificate authority. You will be prompted for a passphrase for the CA private key. Pick a good one and remember it; without the passphrase it will be impossible to issue and sign client certificates:
```
docker run --volumes-from ovpn-data --rm -it kylemanna/openvpn ovpn_initpki
```

**Note, the security of the ovpn-data container is important**. It contains all the private keys to impersonate the server and all the client certificates. Keep this in mind and control access as appropriate. The default OpenVPN scripts use a passphrase for the CA key to increase security and prevent issuing bogus certificates.

## Launch the OpenVPN Server

For easy start/stop/autostart the Docker container that runs the OpenVPN server we will create systemd service config with `nano` or `vim`:
```
sudo vim /etc/systemd/system/docker.openvpn.service
```
Content of the config file:
```
[Unit]
Description=Docker container for OpenVPN server
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=10
Restart=always
ExecStartPre=-/usr/bin/docker stop %n
ExecStartPre=-/usr/bin/docker rm %n
ExecStart=/usr/bin/docker run --volumes-from ovpn-data --rm --name %n -p 1194:1194/udp --
cap-add=NET_ADMIN kylemanna/openvpn

[Install]
WantedBy=multi-user.target
```
Start the OpenVPN server using the systemctl:
```
systemctl start docker.openvpn
```
To enable autostart your OpenVPN server on system boot:
```
systemctl enable docker.openvpn
```
To check all docker container:
```
docker ps -a
```
Output:
```
CONTAINER ID  IMAGE              COMMAND     CREATED        STATUS       PORTS                   NAMES
2a51cc30b13c  kylemanna/openvpn  "ovpn_run"  43 hours ago   Up 43 hours  0.0.0.0:1194->1194/udp  docker.openvpn.service
36f728b5a9f7  busybox            "sh"        44 hours ago   Exited (0)   44 hours ago            ovpn-data
```

## Generate Client Certificates and Config Files

In this section we'll create a client certificate using the PKI CA we created in the last step. Be sure to replace CLIENTNAME as appropriate. The client name is used to identify the machine the OpenVPN client is running on (e.g., "home-laptop", "work-laptop", "nexus5", etc.).

The easyrsa tool will prompt for the CA password. This is the password we set above during the ovpn_initpki command. Create the client certificate:
```
docker run --volumes-from ovpn-data --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass
```
After each client is created, the server is ready to accept connections.

The clients need the certificates and a configuration file to connect. The embedded scripts automate this task and enable the user to write out a configuration to a single file that can then be transfered to the client. Again, replace CLIENTNAME as appropriate:
```
docker run --volumes-from ovpn-data --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
```
The resulting CLIENTNAME.ovpn file contains the private keys and certificates necessary to connect to the VPN. **Keep these files secure and not lying around**. You'll need to securely transport the \*.ovpn files to the clients that will use them. Avoid using public services like email or cloud storage if possible when transferring the files due to security concerns.

Recommend methods of transfer are ssh/scp, HTTPS, USB, and microSD cards where available. For example, using secure copy on your machine:
```
scp user_name@server_ip:/home/user_name/CLIENTNAME.ovpn .
```

## Set Up OpenVPN Clients

The following are commands or operations run on the clients that will connect to the OpenVPN server configured above.

### Ubuntu

Install OpenVPN and Network Manager plugin:
```
sudo apt-get install openvpn network-manager-openvpn network-manager-openvpn-gnome
```
Reboot networking:
```
sudo /etc/init.d/networking restart
sudo /etc/init.d/network-manager restart
```
Setup VPN connection with Network Manager.

### MacOS

Download and install **TunnelBlick**.
```
brew cask install tunnelblick
```
Import the configuration by double clicking the \*.ovpn file copied earlier. TunnelBlick will be invoked and the import the configuration.

Open TunnelBlick, select the configuration, and then select connect.

### Android

Install the **OpenVPN Connect** app from the Google Play store.

Copy CLIENTNAME.ovpn from the server to the Android device in a secure manner. USB or microSD cards are safer. Place the file on your SD card to aid in opening it.

Import the configuration: Menu -> Import -> Import Profile from SD card

Select connect.

### iOS

Install the **OpenVPN Connect** app from the App Store.
