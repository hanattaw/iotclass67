# Install Server and Docker


## How to install Server
* install Ubuntu 24.04 (LTS) on server
* install docker and docker compose
* install wireless-tools, net-tools, git, vim
``` bash
    # install wireless-tools, net-tools, git, vim
    $ sudo apt install wireless-tools net-tools git vim

    # git clone project
    $ git clone https://github.com/sergio11/iot_event_streaming_architecture.git
```
* set up Docker and Run
``` bash
    # use vim edit file
    $ sudo vim /etc/docker/daemon.json

    # Put this text in the file.
    {
      "dns": ["8.8.8.8", "8.8.4.4"]
    }

    # restart service docker
    $ sudo systemctl restart docker

    # docker login
    $ sudo docker login

    # into project and docker compose
    $ cd iot_event_streaming_architecture
    $ sudo docker compose up
```




## How to install Docker

``` bash
# Add Docker's official GPG key:
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update

# Install the Docker packages:
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## How to Uninstall Docker Engine
``` bash
# Uninstall the Docker Engine, CLI, containerd, and Docker Compose packages:
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# Images, containers, volumes, or custom configuration files on your host aren't automatically removed. To delete all images, containers, and volumes:
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
