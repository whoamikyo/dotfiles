My Windows 10 Setup & Dotfiles
==============================

Goals of this setup
-------------------

- Working on Windows 11, on WSL 2 filesystem
- Having a visually nice terminal (Windows Terminal)
- zsh as my main shell
- Using Docker and Docker Compose directly from zsh
- Using IntelliJ IDEA directly from WSL 2


What's in this setup?
---------------------

- Host: Windows 10 2004+
  - Ubuntu/Debian via WSL 2 (Windows Subsystem for Linux)
  - Arch/Manjaro via WSL 2 (Windows Subsystem for Linux)
  - Alpine via WSL 2 (Windows Subsystem for Linux)
- Terminal: Windows Terminal
- [Systemd (Genie, Distrod)](#Systemd)
- zsh + powerlevel10k
- git
- Docker
- Docker Compose
- Node.js (using [Volta](https://volta.sh))
  - node
  - npm
  - yarn
- Go
- IDE: IntelliJ IDEA, under WSL 2, used on Windows via VcXsrv
- WSL Bridge: allow exposing WSL 2 ports on the network


Other guides
------------

- [GitBash with zsh](git-bash/README.md), for better git performances on Windows filesystem
- [Server](server/README.md)


----------------------


Setup WSL 2
-----------

- Enable WSL 2 and update the linux kernel ([Source](https://docs.microsoft.com/en-us/windows/wsl/install-win10))

```powershell
# In PowerShell as Administrator

# Enable WSL and VirtualMachinePlatform features
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Download and install the Linux kernel update package
$wslUpdateInstallerUrl = "https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi"
$downloadFolderPath = (New-Object -ComObject Shell.Application).NameSpace('shell:Downloads').Self.Path
$wslUpdateInstallerFilePath = "$downloadFolderPath/wsl_update_x64.msi"
$wc = New-Object System.Net.WebClient
$wc.DownloadFile($wslUpdateInstallerUrl, $wslUpdateInstallerFilePath)
Start-Process -Filepath "$wslUpdateInstallerFilePath"

# Set WSL default version to 2
wsl --set-default-version 2
```

## Install Distro

```
scoop bucket add whoamikyo https://github.com/whoamikyo/scoop-bucket
```
- scoop install whoamikyo/wsl-ubuntu2004
- scoop install whoamikyo/archwsl
- scoop install whoamikyo/alpinewsl
- scoop install extras/manjarowsl
- scoop install whoamikyo/wsl-debian


Install common dependencies
---------------------------

- Debian/Ubuntu
---------------------------

```shell script
#!/bin/bash

sudo apt update && sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
    git \
    make \
    tig \
    tree \
    zip unzip \
    zsh
```

- Alpine

(TODO)

## Arch/Manjaro

##### Initialize package manager
- Add custom pacman repository with additional packages: 
```
sudo nano /etc/pacman.conf 
```
Then add following to the bottom:
```
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch
```

#### Setting the root password ([Source](https://wsldl-pg.github.io/ArchW-docs/))


```
passwd
```

#### Set up the default user

```
echo "%wheel ALL=(ALL) ALL" > /etc/sudoers.d/wheel

useradd -m -G wheel -s /bin/bash {username}

passwd {username}

exit

Arch.exe config --default-user {username}
```

#### Initialize keyring
Please excute these commands to initialize the keyring. (This step is necessary to use pacman.)
```

sudo pacman-key --init
sudo pacman-key --populate
sudo pacman-key --refresh-keys
sudo pacman -Syy archlinux-keyring
```

Add the line `ParallelDownloads = 5` to:
```
sudo nano /etc/pacman.conf
sudo pacman -Syyuu
sudo pacman -Syyuu --noconfirm \
    curl \
    git \
    zsh \
    bat \
    exa \
    procs \
    bottom \
    make \
    gcc
```

#### Preserving config files from previous WSL installation
Copy .ssh, .kube, .aws from original installation home directory to the Arch WSL2 one.

Fix file permissions for SSH keys, run from the home directory:
```
chmod 0644 .ssh/id_rsa.pub
chmod 0600 .ssh/id_rsa
```

#### Sync pacman and install yaourt:

```
sudo pacman -Syyu && pacman -S yaourt && yaourt -Syyu
```

#### Install yay ([Source](https://github.com/Jguer/yay))

Install dependencies:

```
sudo pacman -S base-devel
```

Choose make and GCC

```
sudo pacman -S --needed --noconfirm git fakeroot && cd /tmp && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
```
#### powerlevel10k ([Source](https://github.com/romkatv/powerlevel10k))

```
yay -S --noconfirm zsh-theme-powerlevel10k-git
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
chsh -s /usr/bin/zsh
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
```

#### zsh-autosuggestions
Clone this repository somewhere on your machine. This guide will assume ~/.zsh/zsh-autosuggestions.

```
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
```
Add the following to your .zshrc:
```
source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
```
Start a new terminal session.


#### Docker

##### Install Kubernetes
```
sudo pacman -S kubectl kubectx
```

#### Install Docker
Uninstall Windows docker to prevent potential naming conflicts.

```
sudo pacman -Syu docker docker-compose
sudo systemctl enable docker     # Start Docker on system startup (optional)
sudo systemctl start docker      # Start Docker now

```
#### Add your user to the `docker` group so you can use Docker without sudo
```
sudo usermod -a -G docker $(whoami)
```

#### Sharing dockerd

choose a common ID for the docker group
If you only plan on using one WSL distro, this next step isn't strictly necessary. However, if you would like to have the option of sharing the Docker socket system-wide, across WSL distributions, then all will need to share a common group ID for the group docker. By default, they each may have a different ID, so a new one is in order.

First, let's pick one. It can be any group ID that is not in use. Choose a number greater than 1000 and less than 65534. To see what group IDs are already assigned that are 1000 or above:
```
getent group | cut -d: -f3 | grep -E '^[0-9]{4}' | sort -g
```
Can't decide what number to use? May I suggest 36257. (Just dial DOCKR on your telephone keypad...) Not likely to be already in use, but check anyway:

```
getent group | grep 36257 || echo "Yes, that ID is free"
```

If the above command returns a line from /etc/group (that does not include docker), then pick another number and try again. If it returns "Yes, that ID is free" then you are good to go, with the following:
```
sudo sed -i -e 's/^\(docker:x\):[^:]\+/\1:36257/' /etc/group
```
Or, if `groupmod` is available (which it is on Fedora, Ubuntu, and Debian, but not Alpine unless you sudo apk add shadow), this is safer:
```
sudo groupmod -g 36257 docker
```
Once the group id has been changed, close the terminal window and re-launch your WSL distro.

*Note that the above steps involving the docker group will need to be run on any WSL distribution you currently have or install in the future, if you want to give it access to the shared Docker socket.*

#### Configure dockerd
Let's first make a shared directory for the docker socket, and set permissions so that the docker group can write to it.
```
DOCKER_DIR=/mnt/wsl/shared-docker
mkdir -pm o=,ug=rwx "$DOCKER_DIR"
chgrp docker "$DOCKER_DIR"
```
I suggest using the configuration file `/etc/docker/daemon.json` to set dockerd launch parameters. If the `/etc/docker` directory does not exist yet, create it with `sudo mkdir /etc/docker/` so it can contain the config file. Then the following, when placed in `/etc/docker/daemon.json`, will set the docker host to the shared socket:
```
{
  "hosts": ["unix:///mnt/wsl/shared-docker/docker.sock"]
}
```
Note: on Debian (but not Fedora or Alpine or Ubuntu), I found that I needed to disable dockerd iptables support so that dockerd would work with the default nftables. I'd love to hear more input on this if anyone has varied experiences with Debian or other distro and nftables/iptables issues with dockerd. To disable iptables support in the config, you can make it look like so:
```
{
  "hosts": ["unix:///mnt/wsl/shared-docker/docker.sock"],
  "iptables": false
}
```
#### Launch dockerd
Most Linux distributions use systemd or other init system, but WSL has its own init system. Rather than twist things to use the existing init system, we just launch dockerd directly:
```
sudo dockerd
```
There should be several lines of info, warnings related to cgroup blkio, and the like, with something like `API listen` on `/mnt/wsl/shared-docker/docker.sock` at the end. If so, you have success.

Open another wsl terminal, and try something like:
```
docker -H unix:///mnt/wsl/shared-docker/docker.sock run --rm hello-world
```
You should see the "Hello from Docker!" message.

#### Launch script for dockerd

The following lines can be placed in .bashrc or .profile if autolaunching is desired, or in a separate shell script. For instance, you may want to create a script `~/bin/docker-service` so that you can run docker-service only when you want, manually.
```
DOCKER_DISTRO="Debian"
DOCKER_DIR=/mnt/wsl/shared-docker
DOCKER_SOCK="$DOCKER_DIR/docker.sock"
export DOCKER_HOST="unix://$DOCKER_SOCK"
if [ ! -S "$DOCKER_SOCK" ]; then
    mkdir -pm o=,ug=rwx "$DOCKER_DIR"
    chgrp docker "$DOCKER_DIR"
    /mnt/c/Windows/System32/wsl.exe -d $DOCKER_DISTRO sh -c "nohup sudo -b dockerd < /dev/null > $DOCKER_DIR/dockerd.log 2>&1"
fi
```

Note that DOCKER_DISTRO should be set to the distro you want to have running dockerd. If unsure of the name, simply run wsl -l -q from Powershell to see your list of WSL distributions. Pick the right one and set it to DOCKER_DISTRO.

The script above does the following:

- Sets the environment variable `$DOCKER_HOST` to point to the shared socket. This isn't necessary for `dockerd` but it will allow running docker without needing to specify `-H unix:///mnt/wsl/shared-docker/docker.sock` each time.
- Checks if the docker.sock file already exists and is a socket. If it is present, do nothing. If not present then the script does the remaining steps, as follows.
- Creates the shared docker directory for the socket and `dockerd` logs, setting permissions appropriately
- Runs `dockerd` from the specified distro. This is important, because it allows any WSL distro to launch dockerd if it isn't already running.
- When `dockerd` is launched, pipe its output and errors to a shared log file.
- When `dockerd` is launched, it runs in the background so you don't have to devote a terminal window to the process as we did earlier. The sudo -b flag gives us this, and we run with nohup so that it runs independent of the terminal, with an explicit null input to nohup to avoid extra warnings. Both standard output and errors are written to the logfile, hence the 2>&1 redirect.

##### Passwordless launch of dockerd
If the above script is placed in .bashrc (most Linux distros) or .profile (distros like Alpine that have Ash/Dash as the default shell), or other shell init script, then it has an unfortunate side effect: you will likely be prompted for a password most every time a new terminal window is launched.

To work around this, you can, if you choose, tell sudo to grant passwordless access to dockerd, as long as the user is a member of the docker group. To do so, enter sudo visudo and add the following line (if your visudo uses vi or vim, then be sure to press "i" to begin editing, and hit ESC when done editing):

```
%docker ALL=(ALL)  NOPASSWD: /usr/bin/dockerd
```
Save and exit ("`:wq`" if the editor is `vi`, or Ctrl-x if it is `nano`), and then you can test if sudo dockerd prompts for a password or not. For peace of mind, you can double-check: something like `sudo -k ls -a /root` should still require a password, unless the password has been entered recently

#### Make sure `$DOCKER_HOST` is always set
If using the script earlier to launch dockerd, then $DOCKER_HOST will be set, and future invocations of docker will not need the unwieldy `-H unix:///mnt/wsl/shared-docker/docker.sock`

If that script is already in your .bashrc or .profile, then the following is unnecessary. If, however, you manually invoke dockerd in some way, then the following may be desireable in your .bashrc or .profile:
```
DOCKER_SOCK="/mnt/wsl/shared-docker/docker.sock"
test -S "$DOCKER_SOCK" && export DOCKER_HOST="unix://$DOCKER_SOCK"
```
The above checks for the docker socket in `/mnt/wsl/shared-docker/docker.sock` and, if present, sets the `$DOCKER_HOST` environment variable accordingly. If you want a more generalized "if this is wsl, then set the socket pro-actively" then you may prefer the following, which simply check for the existence of a `/mnt/wsl` directory and sets the docker socket if so:
```
[ -d /mnt/wsl ] && export DOCKER_HOST="unix:///mnt/wsl/shared-docker/docker.sock"
```
#### Running docker from Windows
If configured as above, I recommend always running docker from wsl. Do you want to run a container? Do so from a WSL window.

This doesn't just apply to the terminal, either. For instance, VSCode supports docker in WSL 2.

But if you want the convenience and utility of running docker in a Powershell window, I have a couple suggestions.

One is to expose dockerd over a TCP Port. That method will be explored in a later.

A simplified method I recommend: a Powershell function that calls the WSL docker, passing along any arguments. This function can be placed in your Powershell profile, usually located at `~\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1`
```
$DOCKER_DISTRO = "Arch"
function docker {
    wsl -d $DOCKER_DISTRO docker -H unix:///mnt/wsl/shared-docker/docker.sock @Args
}
```
Again, try `wsl -l -q` to see a list of your WSL distributions if you are unsure which one to use.

Make sure the Docker daemon is running, then launch a new Powershell window, and try the hello-world container again. Success?

You could also make a batch file with the appropriate command in it. For instance, name it `docker.bat` and place in `C:\Windows\system32` or other location included in `%PATH%`. The following contents will work in such a script:
```
@echo off
set DOCKER_DISTRO=fedora
wsl -d %DOCKER_DISTRO% docker -H unix:///mnt/wsl/shared-docker/docker.sock %*
```
You could go a step further and ensure that dockerd is running whenever you start Powershell. Assuming that the dockerd start script detailed above is saved in a file in WSL as `$HOME/bin/docker-service` and is executable (try `chmod a+x $HOME/bin/docker-service`), then the following line in your Powershell profile will launch dockerd automatically:
```
wsl -d $distro ~/bin/docker-service
```
If you don't want to rely on a particular WSL shell script, you could implement a Powershell function to launch dockerd, such as this:
```
function Docker-Service {
  Param ([string]$distro)
  $DOCKER_DIR = "/mnt/wsl/shared-docker"
  $DOCKER_SOCK = "$DOCKER_DIR/docker.sock"
  wsl -d "$distro" sh -c "[ -S '$DOCKER_SOCK' ]"
  if ($LASTEXITCODE) {
    wsl -d "$distro" sh -c "mkdir -pm o=,ug=rwx $DOCKER_DIR ; chgrp docker $DOCKER_DIR"
    wsl -d "$distro" sh -c "nohup sudo -b dockerd < /dev/null > $DOCKER_DIR/dockerd.log 2>&1"
  }
}
```
This function takes one parameter: the distro name.

In all of the above, the principle is the same: you are launching Linux executables, using WSL [interoperability](https://docs.microsoft.com/en-us/windows/wsl/interop).

#### A note on bind mounts: stay on the Linux filesystem
With docker, it is possible to mount a host system's directory or files in the container. The following often works, but is not advisable when launching WSL docker from Windows:
```
echo "<h1>Hello, World</h1>" > index.html
docker run -p "8080:80" -v "$PWD:/usr/share/nginx/html:ro" nginx
```
Instead of doing the above haphazardly, when launching WSL docker from Powershell, two recommendations:

- For performance reasons, only bind mount from within the Linux filesystem. To get to a Linux directory while in Powershell, try something like `cd (wsl wslpath -m ~/path/to/my/dir)`
- Instead of `$PWD` to get the current directory, try `'$(wslpath -a .)'` and, yeah, the single quotes are helpful for escaping.
An example:
```
cd (wsl wslpath -m ~)
mkdir html
cd html
echo "<h1>Hello, World</h1>" > index.html
docker run -p "8080:80" -v '$(wslpath -a .):/usr/share/nginx/html:ro' nginx
```
Then point your browser to [http://localhost:8080](http://localhost:8080), and happiness will result. (Depending on your network [configuration](https://docs.microsoft.com/en-us/windows/wsl/compare-versions#accessing-a-wsl-2-distribution-from-your-local-area-network-lan), you may instead need to access this through `http://[WSL IP Address]:`8080` which should be obtainable with ifconfig or ip addr)

Try `wsl wslpath` from Powershell, or just `wslpath` from Linux, to see the options. This is a very useful tool, to say the least.

#### Example scripts

Some of the code examples above have been placed in scripts folder:

- **docker-service.sh** is a Unix shell script to launch `dockerd`
- **docker-service.ps1** contains a Powershell function to launch `dockerd` in WSL
- **docker.bat** is a Windows batch file for launching WSL `docker` from CMD
- **docker.ps1** contains a Powershell function for launching WSL `docker` from Powershell, if placed in your profile

No doubt there are ways these can be tweaked to be more useful and reliable; feel free to post in the comments.

------
Additionally, you can run Docker by installing the Docker binary  sudo pacman -S docker , but in order to use Docker in WSL, you need to have Docker installed in Windows and expose the Docker API to `tcp://0.0.0.0:2375`.

In .bashrc, add the below entry to allow Docker from WSL  to access Windows Docker.

```
export DOCKER_HOST=tcp://0.0.0.0:2375
```

#### Install programming languages
```
sudo pacman -S ruby nodejs python go crystal php jre-openjdk-headless
```
#### Installing protocol buffers Main binaries/libraries: 
```
sudo pacman -S protobuf protobuf grpc grpc-cli
```
#### gRPC for Python and PHP: 
```
sudo pacman -S python-grpcio php-grpc
```
#### gRPC & Protobuf for Go: 
```
yay -S protobuf-go protoc-gen-go-grpc
```
#### gRPC & Protobuf for Ruby: 
```
gem install google-protobuf grpc grpc-tools
```

#### Optional: Neo Vim + Lunar Vim ([Source](https://github.com/LunarVim/LunarVim))


-------
```
sudo pacman -S neovim
sudo pacman -S yarn npm rust base-devel

bash <(curl -s https://raw.githubusercontent.com/lunarvim/lunarvim/master/utils/installer/install.sh)
export PATH=~/.cargo/bin:~/.local/bin:$PATH
lvim

```
Add to PATH
```
nano ~/.zshrc
export PATH=$HOME/.local/bin:$HOME//.cargo/bin:$PATH
```

#### edit config.lua

```
vim.opt.timeoutlen = 500
```

GPG key
-------

If you already have a GPG key, restore it. If you did not have one, you can create one.

### Restore

- On old system, create a backup of a GPG key
  - `gpg --list-secret-keys`
  - `gpg --export-secret-keys {{KEY_ID}} > /tmp/private.key`
- On new system, import the key:
  - `gpg --import /tmp/private.key`
- Delete the `/tmp/private.key` on both side

### Create

- `gpg --full-generate-key`

[Read GitHub documentation about generating a new GPG key for more details](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-gpg-key).


Setup Git
---------

```shell script
#!/bin/bash

# Set username and email for next commands
email="erikferreira@outlook.com"
username="Erik Ferreira"
gpgkeyid="8FA78E6580B1222A"

# Configure Git
git config --global user.email "${email}"
git config --global user.name "${username}"
git config --global user.signingkey "${gpgkeyid}"
git config --global commit.gpgsign true
git config --global core.pager /usr/bin/less
git config --global core.excludesfile ~/.gitignore

# Generate a new SSH key
ssh-keygen -t rsa -b 4096 -C "${email}"

# Start ssh-agent and add the key to it
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_rsa

# Display the public key ready to be copy pasted to GitHub
cat ~/.ssh/id_rsa.pub
```

- [Add the generated key to GitHub](https://github.com/settings/ssh/new)


Setup zsh
---------

```shell script
#!/bin/bash

# Clone the dotfiles repository
mkdir -p ~/dev/dotfiles
    git clone git@github.com:whoamikyo/dotfiles.git ~/dev/dotfiles

# Install Antibody and generate .zsh_plugins.zsh
curl -sfL git.io/antibody | sudo sh -s - -b /usr/local/bin
antibody bundle < ~/dev/dotfiles/zsh_plugins > ~/.zsh_plugins.zsh

# Link custom dotfiles
ln -sf ~/dev/dotfiles/.aliases.zsh ~/.aliases.zsh
ln -sf ~/dev/dotfiles/.p10k.zsh ~/.p10k.zsh
ln -sf ~/dev/dotfiles/.zshrc ~/.zshrc
ln -sf ~/dev/dotfiles/.gitignore ~/.gitignore

# Create .screen folder used by .zshrc
mkdir ~/.screen && chmod 700 ~/.screen

# Change default shell to zsh
chsh -s $(which zsh)
```


Setup GitHub CLI
----------------

```shell script
#!/bin/zsh

curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh
```

Login gh command to GitHub via `gh auth login`


#### Systemd
-------
#### Option 1: This uses [arkane-systems/genie](https://github.com/arkane-systems/genie).



```shell script
#!/bin/zsh

# Setup Microsoft repository (Genie depends on .NET)
curl -sL -o /tmp/packages-microsoft-prod.deb "https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb"
sudo dpkg -i /tmp/packages-microsoft-prod.deb
rm -f /tmp/packages-microsoft-prod.deb

# Setup Arkane Systems repository
sudo curl -sL -o /usr/share/keyrings/wsl-transdebian.gpg https://arkane-systems.github.io/wsl-transdebian/apt/wsl-transdebian.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/wsl-transdebian.gpg] https://arkane-systems.github.io/wsl-transdebian/apt/ \
  $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/wsl-transdebian.list > /dev/null

# Install Systemd Genie
sudo apt-get update
sudo apt-get install -y systemd-genie

# Mask some unwanted services
sudo systemctl mask systemd-remount-fs.service
sudo systemctl mask multipathd.socket

# Install custom config
sudo ln -sf ~/dev/dotfiles/genie.ini /etc/genie.ini
```

#### Option 2: This uses [nullpo-head/wsl-distrod](https://github.com/nullpo-head/wsl-distrod).

1. Download and run the latest installer script.

   ```bash
   curl -L -O "https://raw.githubusercontent.com/nullpo-head/wsl-distrod/main/install.sh"
   chmod +x install.sh
   sudo ./install.sh install
   ```

   This script installs distrod, but doesn't enable it yet.

2. Enable distrod in your distro

   You have two options.
   If you want to automatically start your distro on Windows startup, enable distrod by the following command

   ```bash
   /opt/distrod/bin/distrod enable --start-on-windows-boot
   ```

   Otherwise,

   ```bash
   /opt/distrod/bin/distrod enable
   ```

   You can run `enable` with `--start-on-windows-boot` again if you want to enable autostart later.

3. Restart your distro

   After re-opening a new WSL window, your shell runs in a systemd session.
   
   ## Update Distrod

1. Inside a Distrod session, download and run the latest installer script.

   ```bash
   curl -L -O "https://raw.githubusercontent.com/nullpo-head/wsl-distrod/main/install.sh"
   chmod +x install.sh
   sudo ./install.sh update
   ```

## How Distrod Works

In a nutshell, Distrod is a binary that creates a simple container that runs systemd as an init process,
and starts your WSL sessions within that container. To realize that, Distrod does the following things.

- Modify the rootfs of the concrete distro you chose so that it is compatible with both WSL and systemd.
  - Modify systemd services so that they are compatible with WSL
  - Configure networks for WSL
  - Put `/opt/distrod/bin/distrod` and other resources in the rootfs.
  - Register the Distrod's binary as the login shell
- When Distrod is launched by WSL's init as a login shell, Distrod
  1.  Starts systemd in a simple container
  2.  Launches your actual shell within that container
  3.  Bridges between the systemd sessions and the WSL interop environment.

Docker
---------------------------

#### Debian/Ubuntu

```shell script
#!/bin/zsh

# Add Docker to sources.list
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install tools
sudo apt update && sudo apt install -y \
    docker-ce docker-ce-cli containerd.io

# Add user to docker group
sudo usermod -aG docker $USER
```

You can start Docker daemon via [Systemd](#systemd) or via `dcs` alias.


Docker Compose
--------------

```shell script
#!/bin/zsh

sudo curl -sL -o /usr/local/bin/docker-compose $(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep "browser_download_url.*$(uname -s | awk '{print tolower($0)}')-$(uname -m)" | grep -v sha | cut -d: -f2,3 | tr -d \")
sudo chmod +x /usr/local/bin/docker-compose
```

#### Alpine

(TODO)

#### Arch/Manjaro

(TODO)

---------------------------

Node.js
-------

```shell script
#!/bin/zsh

# Install Volta
mkdir -p $VOLTA_HOME
curl https://get.volta.sh | bash -s -- --skip-setup

# Install node and package managers
volta install node npm yarn
```


Go
---

```shell script
#!/bin/zsh

goVersion=1.16.4
curl -L "https://golang.org/dl/go${goVersion}.linux-amd64.tar.gz" > /tmp/go${goVersion}.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf /tmp/go${goVersion}.linux-amd64.tar.gz
rm /tmp/go${goVersion}.linux-amd64.tar.gz
```

[See official documentation](https://golang.org/doc/install)


IntelliJ IDEA
-------------

I run IntelliJ IDEA in WSL 2, and get its GUI on Windows via X Server (VcXsrv).

### Setup VcXsrv

- [Install VcXsrv (XLaunch)](https://sourceforge.net/projects/vcxsrv/)

```shell script
windowsUserProfile=/mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')

# Run VcXsrv at startup
cp ~/dev/dotfiles/config.xlaunch "${windowsUserProfile}/AppData/Roaming/Microsoft/Windows/Start Menu/Programs/Startup"
```

### Install IntelliJ IDEA

```shell script
#!/bin/zsh

# Install IDEA dependencies
sudo apt update && sudo apt install -y \
    fontconfig \
    libxss1 \
    libnss3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libgbm1 \
    libpangocairo-1.0-0 \
    libcups2 \
    libxkbcommon0

# Create install folder
sudo mkdir /opt/idea

# Allow your user to run IDEA updates from GUI
sudo chown $UID:$UID /opt/idea

# Download IntelliJ IDEA
curl -L "https://download.jetbrains.com/product?code=IIU&latest&distribution=linux" | tar vxz -C /opt/idea --strip 1
```


Setup Windows Terminal
----------------------

- scoop install JetBrains-Mono
- scoop install whoamikyo/windows-terminal-preview

```shell script
#!/bin/zsh

windowsUserProfile=/mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')

# Copy Windows Terminal settings
cp ~/dev/dotfiles/terminal-settings.json ${windowsUserProfile}/AppData/Local/Packages/Microsoft.WindowsTerminal_8wekyb3d8bbwe/LocalState/settings.json
```


WSL Bridge
----------

When a port is listening from WSL 2, it cannot be reached.
You need to create port proxies for each port you want to use.
To avoid doing than manually each time I start my computer, I've made the `wslb` alias that will run the `wsl2bridge.ps1` script in an admin Powershell.

```shell script
#!/bin/zsh

windowsUserProfile=/mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')

# Get the hacky network bridge script
cp ~/dev/dotfiles/wsl2-bridge.ps1 ${windowsUserProfile}/wsl2-bridge.ps1
```

In order to allow `wsl2-bridge.ps1` script to run, you need to update your PowerShell execution policy.

```powershell
# In PowerShell as Administrator

Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
PowerShell -File $env:USERPROFILE\\wsl2-bridge.ps1
```

Then, when port forwarding does not work between WSL 2 and Windows, run `wslb` from zsh:

```shell script
#!/bin/zsh

wslb
```

Note: This is a custom alias. See [`.aliases.zsh`](.aliases.zsh) for more details


Limit WSL 2 RAM consumption
---------------------------

```shell script
#!/bin/zsh

windowsUserProfile=/mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')

# Avoid too much RAM consumption
cp ~/dev/dotfiles/.wslconfig ${windowsUserProfile}/.wslconfig
```

Note: You can adjust the RAM amount in `.wslconfig` file. Personally, I set it to 8 GB.


Install kubectl
---------------

```shell script
#!/bin/zsh

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

[Original documentation](https://master--kubernetes-io-master-staging.netlify.app/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)


Install AWS CLI
---------------

```shell script
#!/bin/zsh

cd /tmp/aws-cli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
cd -
rm -rf /tmp/aws-cli
```

[Original documentation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)


Setup Git Filter Repo
---------------------

```shell script
#!/bin/zsh

git clone git@github.com:newren/git-filter-repo.git /tmp/git-filter-repo
cd /tmp/git-filter-repo
make snag_docs
sudo cp -a git-filter-repo $(git --exec-path)
sudo cp -a Documentation/man1/git-filter-repo.1 $(git --man-path)/man1
sudo cp -a Documentation/html/git-filter-repo.html $(git --html-path)
cd -
rm -rf /tmp/git-filter-repo
```
