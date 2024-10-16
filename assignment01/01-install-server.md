# Install Server and Docker

## How to install Server 🖥️
### Install Ubuntu 24.04 (LTS) on server

#### 1. Select Your Language
Choose your preferred language.
  - Example: `[ English ]`

#### 2. Installer Update Available
Skip the update for now.
  - Select: `[ Continue without updating ]`

#### 3. Keyboard Configuration
If the default keyboard layout is correct, proceed.
  - Select: `[ Done ]`

#### 4. Choose the Type of Installation
Choose the minimized Ubuntu Server installation.
  - Option: `(X) Ubuntu Server (minimized)`
  - Select: `[ Done ]`

#### 5. Network Configuration
Accept the default network configuration.
  - Select: `[ Done ]`

#### 6. Proxy Configuration
If you don't use a proxy, leave this blank.
  - Select: `[ Done ]`

#### 7. Ubuntu Archive Mirror Configuration
Accept the default mirror settings.
  - Select: `[ Done ]`

#### 8. Guided Storage Configuration
Choose to create a custom storage layout.
  - Option: `(X) Custom storage layout`
  - Select: `[ Done ]`

#### 9. Storage Configuration
Configure your partitions as follows:

1. **Mount Boot Partition:**
   - Use free space to create a new GPT partition.
   - Size: `2G`
   - Format: `[ ext4 ]`
   - Mount: `[ /boot ]`
   - Select: `[ Create ]`

2. **Mount Swap Partition:**
   - Use free space to create a new GPT partition.
   - Size: `2G`
   - Format: `[ swap ]`
   - Select: `[ Create ]`

3. **Mount Root Partition:**
   - Use the remaining free space to create a new GPT partition.
   - Size: `MAX`
   - Format: `[ ext4 ]`
   - Mount: `[ / ]`
   - Select: `[ Create ]`

#### **File System Summary:**

| MOUNT POINT | SIZE  | TYPE | DEVICE TYPE            |
|-------------|-------|------|------------------------|
| /           | MAX   | ext4 | partition of localdis  |
| /boot       | 2.0 G | ext4 | partition of localdis  |
| /SWAP       | 2.0 G | swap | partition of localdis  |

- Select: `[ Done ]`
- Confirm the destructive action.
  - Select: `[ Continue ]`

#### 10. Profile Configuration
Fill in the following fields to set up your server profile:

- **Your name:** Enter your full name.
- **Your server name:** Choose a name for your server.
- **Pick a username:** Choose a username for your account.
- **Choose a password:** Create a secure password.
- **Confirm your password:** Re-enter your password to confirm.
- Select: `[ Done ]`

#### 11. Upgrad to Ubunto Pro
If prompted, select the option for upgrading to Ubuntu Pro.
- Option: `(X) Custom storage layout`
- Select: `[ Continue ]`

#### 12. SSH Configuration
Install the OpenSSH server to enable remote access.
- Option: `(X) Install OpenSSH server`
- Select: `[ Done ]`

#### 13. Featured server snaps
Skip the installation of featured snaps.
- Select: `[ Continue ]`

#### 14. Installation Complete!
Once the installation is finished, your server is ready to use.
- Select: `[ Reboot Now ]`

---



### Setup Basic tools and Clone Project
``` bash
    # install wireless-tools, net-tools, git, vim
    sudo apt install wireless-tools net-tools git vim

    # git clone project
    git clone https://github.com/sergio11/iot_event_streaming_architecture.git

    # into project 
    cd iot_event_streaming_architecture

```

## How to install Docker 🐳

``` bash
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    # Install the Docker packages:
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
### Manage Docker as a non-root user
To create the `docker` group and add your user:
1. Create the `docker` group.
``` bash
sudo groupadd docker
```
2. Add your user to the `docker` group.
```bash
sudo usermod -aG docker $USER
```
3. Log out and log back in so that your group membership is re-evaluated.
```bash
newgrp docker
```

### Set up DNS Docker and Docker login
``` bash
    # use vim edit file
    sudo vim /etc/docker/daemon.json

    # Put this text in the file.
    {
      "dns": ["8.8.8.8", "8.8.4.4"]
    }

    # restart service docker
    sudo systemctl restart docker

    # docker login
    sudo docker login
```

## How to Uninstall Docker Engine
``` bash
    # Uninstall the Docker Engine, CLI, containerd, and Docker Compose packages:
    sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

    # Images, containers, volumes, or custom configuration files on your host aren't automatically removed. To delete all images, containers, and volumes:
    sudo rm -rf /var/lib/docker
    sudo rm -rf /var/lib/containerd
```
