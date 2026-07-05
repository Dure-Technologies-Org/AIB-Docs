# Setup

## CUDA related

Fetch latest upgrades: `sudo apt update && sudo apt upgrade`  

Then update the drivers: `sudo ubuntu-drivers install`  

Reboot jetson: `sudo reboot`

for **torchcodec**:
```bash
sudo apt-get update
sudo apt-get install -y libavformat-dev libavcodec-dev libavutil-dev libavdevice-dev libavfilter-dev
```

install **torchaudio**:
```bash
sudo apt-get update
sudo apt-get install -y ffmpeg libavformat-dev libavcodec-dev libavutil-dev libavdevice-dev libavfilter-dev libswresample-dev libswscale-dev cmake ninja-build
sudo apt install libnccl2 libnccl-dev
```

for **torch**:  
Install `libcudss.so.0` and register its path for LD_LIBRARY_PATH:
```bash
sudo apt-get install -y libcudss0-cuda-12
echo "/usr/lib/aarch64-linux-gnu/libcudss/12" | sudo tee /etc/ld.so.conf.d/libcudss-12.conf 
sudo ldconfig
```

## Database

We will install postgres-16 for our work:  
1. Check ubuntu version: `lsb_release -a`, lets say its `jammy`
2. Install the repo key: 
    ```bash
    sudo apt install curl ca-certificates
    sudo install -d /usr/share/postgresql-common/pgdg
    sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
    ```
3. Create `/etc/apt/sources.list.d/pgdg.sources`, for `jammy`, download `jammy-pgdg`:
    ```bash
    Types: deb deb-src
    URIs: https://apt.postgresql.org/pub/repos/apt
    Suites: jammy-pgdg
    Architectures: arm64
    Components: main
    Signed-By: /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
    ```
4. Install postgresSQL 16:
    ```bash
    sudo apt update
    sudo apt install postgresql-16
    ```
5. Install postgresSQL 16 extensions:
    ```bash
    sudo apt install postgresql-16-pgvector
    ```
6. Start and enable services:
    ```bash
    sudo systemctl start postgresql
    sudo systemctl enable postgresql
    sudo systemctl status postgresql
    ```
Optional: In case jetson had older postgreSQL server installed, above steps will only update the psql client and download v16 server packages (check `pg_lsclusters`). To update the server too, perform migration of the cluster by doing: `sudo pg_upgradecluster 14 main`

We will now backup and restore the UAT version of database from main server to jetson:
1. On jetson, create an empty db: 
    ```bash
    sudo -u postgres createdb -T template0 intellicare_uat
    ```
2. Create the dev user and give it access to this db: 
    ```bash
    CREATE ROLE intellicare_dev_usr
    LOGIN PASSWORD 'your_password_here';

    ALTER ROLE intellicare_dev_usr CREATEDB;

    ALTER DATABASE intellicare_uat OWNER TO intellicare_dev_usr;
    ```

3. On main servers:
    ```bash
    sudo -u postgres pg_dump uat_db_name > uat_db_name.dump
    rsync -avz /path/to/uat_db_name.dump jetson_user@jetson_hostname:/path/to/jetson/dir/
    ```
4. Then on jetson:
    ```bash
    sudo -u postgres psql intellicare_uat < uat_db_name.dump
    ```

## Storage

### Project Mount Point

Rename any mount points `/mnt/ssd` to `/idata`:  
```bash
sudo mkdir /idata
sudo umount /mnt/ssd
``` 

Make sure that `lsblk` command shows blank under `MOUNTPOINTS` column.
If it doesn't, perform a lazy unmount and check `lsblk` again:  
```bash
sudo umount -l /mnt/ssd
```

Edit `/etc/fstab` and rename `/mnt/ssd` to `/idata` and then mount `/idata`:  
```bash
sudo mount /idata
```

Clear the system cache and reload:  
```bash
sudo systemctl daemon-reload
sudo mount -a
```

### HF and UV Cache

In `~/.bashrc` add following lines:  
```bash
export HF_HOME=/idata/.cache/huggingface
export UV_CACHE_DIR=/idata/.cache/uv
```
Refresh your terminal : `source ~/.bashrc`

Create home and cache dirs in `/idata` for HF and UV:  
```bash
mkdir -p $HF_HOME
mkdir -p $UV_CACHE_DIR
```

## User and group management

Create a new group:  
```bash
sudo groupadd jetson-devs
```

Create a new user and give sudo rights and add it to the dev group:
```bash
sudo adduser teammember-name
sudo usermod -aG sudo,jetson-devs teammember-name
```
You might also want to add the `teammember-name` to other groups like `video`,`i2c` etc. for CUDA, camera etc access.

Change ownership of your project mount folder to this group:  
```bash
sudo chown -R :jetson-devs /idata/ai-in-the-box
```

Set the SGID permission so any new file created by either of you automatically inherits this group ownership:
```bash
sudo chmod -R g+rwXs /idata/ai-in-the-box
```

## Git setup

For a single Jetson, multiple user setup: work in your respective cloned repo folders under `/idata`.

BUT if all still want to work in the same project dir and maintain git: work exclusively/sequentially and not parallely.  


Linux usernames are seperate but git repo will be the same that tracks each user seperately. 
That is why we need to tell git that its safe to have seperate linux users access the same repo:
```bash
git config --global --add safe.directory /idata/intellicare_uat
```
and add a seperate upstream to your respective fork on github:
```bash
git remote add my-fork <YOUR_PERSONAL_FORK_URL>
```
DO NOT blindly do `add origin` otherwise the git will overwrite the `ai-in-the-box` url with your fork url. 

Configure local identity:
```bash
git config --local user.name "Teammate Name"
git config --local user.email "teammate@example.com"
```


## SSH

### Jump host

We can make the server our jumphost to jetson and save the configurations in our local `~/.ssh/config`. This will allow us to directly SSH into jetson:

```bash
Host <server_name>
    Hostname <server IP>
    User <server login username>
    IdentityFile ~/.ssh/id_rsa

Host jetson
    HostName <jetson IP>
    User <jetson login username>
    IdentityFile ~/.ssh/id_rsa
    ProxyJump <server_name>
```
This assumes that you used `id_rsa` as your public keys. Since you copied your public keys in `authorized_keys` on server too, jetson will authenticate you from those keys.

Save the server public keys to jetson `~/.ssh/authorized_keys`, run the following command on the server:
```cmd
ssh-copy-id <jetson login username>@<jetson IP>
```

## Network

### Requesting IP

On jetson, request a new IP from the DHCP server after a fresh reboot: 
```cmd
sudo dhclient eno1
```

Check Tailscale, whether jetson(yahboom user) has been given a new IP: 
![tailscale screenshot](img/tailscale_yahboom.png)

### Autoconnecting to ethernet on device restart

Check if NetworkManager is running: `systemctl status NetworkManager`  
 
If it is not running:  
```bash
sudo systemctl start NetworkManager
sudo systemctl enable NetworkManager
```

If it is already running, find the ethernet connection name: `nmcli connection show`  

The name will likely be `eno1`, then:  
```bash
sudo nmcli connection modify "ethernet_name" connection.autoconnect yes
sudo nmcli connection modify "ethernet_name" ipv4.method auto
```

### Setting DNS

In case of jetson does not reach a DNS server:  

1. Sudo open `/etc/systemd/resolved.conf` in your preferred text editor.  
2. Under the `[Resolve]` section, uncomment and modify the lines:  

```bash
DNS=1.1.1.1 8.8.8.8
FallbackDNS=1.0.0.1 8.8.4.4
```

3. Restart the service: sudo systemctl restart systemd-resolved

Once you've made your changes, test your DNS: `resolvectl status`

## Tools

### jtop

Intall using `pip`: 
```bash
sudo pip3 install jetson-stats
sudo systemctl restart jtop.service
```
Jetson might or might not need a reboot after this.

### dust

Convenient `du` wrapper in rust:  
```bash
curl -sSfL https://raw.githubusercontent.com/bootandy/dust/refs/heads/master/install.sh | sh
```

### fzf

Fuzzy finder for dev productivity:
```bash
wget https://github.com/junegunn/fzf/releases/download/v0.73.1/fzf-0.73.1-linux_arm64.tar.gz
tar -xzf fzf
sudo mv fzf /usr/local/bin/
rm fzf-0.73.1-linux_arm64.tar.gz
sudo chmod a+x /usr/local/bin/fzf
```

For key-bindings to work, perform shell integration by adding this line to `~/.bashrc`:
```bash
eval "$(fzf --bash)"
```

### Docker

Configure docker to run gpu-accelerated containers from nvidia primarily.
Assuming docker, nvidia-container is preinstalled and user is added to docker group, run:  
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Add nvidia as the default runtime for docker:
```bash
sudo vim /etc/docker/daemon.json
```
Edit the file:
```bash
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "default-runtime": "nvidia"
}
```
Now restart:
```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

Now move the docker data dir to `/idata` that has more space:

```bash
sudo du -csh /var/lib/docker/ && \
sudo mkdir /idata/docker && \
sudo rsync -axPS /var/lib/docker/ /idata/docker/ && \
sudo du -csh /idata/docker/
```

We will also move the data dir to `/idata`:
```bash
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo systemctl stop containerd
sudo umount -f /var/lib/docker/overlay2/*/*/merged
sudo umount -f /var/lib/docker/containers/*/mounts/shm
sudo du -csh /var/lib/docker && sudo rsync -axPS /var/lib/docker/ /idata/docker/ && sudo du -csh /idata/docker/
```

In case `rsync` failes and the data that cannot be transferred is very small, ignore those errors by giving flag on the above `rsync` command: `-ignore-errors` and rerun the command.

Edit the `sudo vim /etc/docker/daemon.json` file again:
```bash
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "default-runtime": "nvidia",
    "data-root": "/idata/docker"
}
```

Rename old data dir and restart docker daemon:  
```bash
sudo mv /var/lib/docker /var/lib/docker.old
sudo systemctl daemon-reload && \
sudo systemctl restart docker && \
sudo journalctl -u docker
```

### Ollama

We will install jetson optimized ollama container:  
```bash
docker pull dustynv/ollama:r36.4.3
```

### ffmpeg

```bash
sudo apt-get install -y \
  ffmpeg \
  libavutil-dev libavcodec-dev libavformat-dev libavdevice-dev libavfilter-dev libswresample-dev libswscale-dev
```
