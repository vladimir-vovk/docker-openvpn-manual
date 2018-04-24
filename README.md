# Setup OpenVPN server with docker manual

This manual is based on awesome Kyle Manna's [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) repo and article [How To Run OpenVPN in a Docker Container on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-run-openvpn-in-a-docker-container-on-ubuntu-14-04), but updated for Ubuntu 16.04.

* You need your own VPS with Ubuntu 16.04 and ssh access.
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

