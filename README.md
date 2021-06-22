# Docker-Registry

## Prepare System
- Multiple VM Ubuntu version 20.04 on Hyper-V   [set up hyper-v on windows10](https://github.com/EknarongAphiphutthikul/Hyper-V)
- DNS Server  [set up](https://github.com/EknarongAphiphutthikul/Dns-bind9)
- Update Package On Ubuntu 20.04
  ```sh
  sudo apt-get update
  ```
- Show hostname
  ```sh
  hostnamectl
  ```
- Set hostname
  ```sh
  sudo hostnamectl set-hostname registry.ake.com
  ```
- Show ip
  ```sh
  ifconfig
  ```
- Set ipv4 internal network (vEthernet-Internal-ME)
  - On cloud : you'll need to disable.
    ```sh
    sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
    ```
    ```console
    network: {config: disabled}
    ```
  - Show file config in netplan
    ```sh
    ls /etc/netplan/
    ```
    ```console
    00-installer-config.yaml
    ```
  - Edit file config in netplan
    ```sh
    sudo nano /etc/netplan/00-installer-config.yaml
    ```
    ```console
    network:
      ethernets:
        eth0:
          dhcp4: false
          addresses:
            -  169.254.19.106/16
          nameservers:
            search: [ ake.com ]
            addresses:
              - 169.254.19.105
        eth1:
          dhcp4: true
      version: 2
    ```
  - Apply Config
    ```sh
    sudo netplan apply
    ```

- Set DNS (change default 127.0.0.53 to 169.254.19.105)  
  > **Important** : Workaround for  [Bug #1624320](https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/1624320)
  ```sh
  sudo rm -f /etc/resolv.conf
  sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
  sudo reboot
  ```
----

<br/>

## Install Docker

- Install the required dependencies
  ```sh
  sudo apt-get install apt-transport-https ca-certificates curl software-properties-common curl -y
  ```
- Import the Docker GPG key
  ```sh
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  ```
- Add the Docker CE official repository to the APT source file
  ```sh
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  ```
- Update the repository cache
  ```sh
  sudo apt-get update -y
  ```
- Install
  ```sh
  sudo apt-get install docker-ce -y
  ```
- Verify the installed version of Docker CE
  ```sh
  docker --version
  ```
----

<br/>
  
## Install Docker-Registry
- Pull Image
  ```sh
  sudo docker pull registry:2.7.1
  ```
- Create certificates  
  [How to create certificates](https://github.com/EknarongAphiphutthikul/OpenSSL-Certificate-Authority)  
  copy registry.key.pem, registry.cert.pem and ca-chain.cert.pem to /home/akeadm/certs/
  ```sh
  ls /home/akeadm/certs/
  ```
  ```console
  ca-chain.cert.pem  registry.cert.pem  registry.key.pem
  ```
  decrypt file registry.key.pem :
  ```sh
  openssl rsa -in registry.key.pem -out registry.ake.com.key -passin pass:changeit
  ```

  merge file crt :
  ```sh
  cat registry.cert.pem ca-chain.cert.pem > registry.ake.com.crt
  ```
  delete file : 
  ```sh
  rm registry.key.pem registry.cert.pem ca-chain.cert.pem

  ls /home/akeadm/certs/
  ```
  ```console
  registry.ake.com.crt  registry.ake.com.key
  ```
- Create Native basic auth
  ```sh
  mkdir /home/akeadm/auth

  sudo docker run --entrypoint htpasswd httpd:2 -Bbn registryusr registrypw@ > /home/akeadm/auth/htpasswd
  ```
- Start Docker Registry
  ```sh
  mkdir -p /home/akeadm/registry/data

  sudo docker run -d --restart=always --name registry \
  -v /home/ake/registry/data:/var/lib/registry \
  -v /home/akeadm/certs:/certs \
  -v /home/akeadm/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.ake.com.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.ake.com.key \
  -p 443:443 \
  registry:2.7.1

  sudo docker ps
  ```
  ```console
  CONTAINER ID   IMAGE            COMMAND                  CREATED              STATUS              PORTS                                             NAMES
  2706265cf301   registry:2.7.1   "/entrypoint.sh /etc…"   About a minute ago   Up About a minute   0.0.0.0:443->443/tcp, :::443->443/tcp, 5000/tcp   registry

  ```
----

<br/>

## Test Docker-Registry