# Setup

##  First boot-up

1. Connect monitor, keyboard and mouse via bluetooth or usb-c dongle. Connect ethernet cable.  

2. Admin user name: `dure`

3. Download [Tailscale app](https://tailscale.com/download/mac).   
3.1. Allow it to run in background. Allow it to create VPN.  
3.2. Login to tailscale account and then rename the device to something appropriate.  
3.3. In the Tailsclae app settings, allow to add tailscale to PATH so that it can run as cli.

4. Skip Apple account creation, apple intelligence, FileVault, and other prompts.

5. Create a new user `intellicare` as `Standard` type.  
5.1. Name its home folder as the user name ie. `intellicare`.  
5.2. Make this user as the auto-login user. So that on re-boots this user logs in.

6. Under Energy mode settings:  
6.1. prevent automatic sleeping when display is off.  
6.2. Restart after power failure

7. Go to Safari and type: dure-technologies-org.github.io/AIB-Docs/macmini/setup/  

## Tailscale 

Install using brew. Homebrew installation will take a few minutes and your admin password:  
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install --formula tailscale
```
```bash
sudo nano /Library/LaunchDaemons/com.tailscale.tailscaled.plist
```

Edit the plist as following:  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://apple.com">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.tailscale.tailscaled</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Applications/Tailscale.app/Contents/MacOS/Tailscale</string>
        <string>--be-headless</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Set the permissions, load and verify it:  
```bash
sudo chown root:wheel /Library/LaunchDaemons/com.tailscale.tailscaled.plist
sudo chmod 644 /Library/LaunchDaemons/com.tailscale.tailscaled.plist
sudo launchctl load -w /Library/LaunchDaemons/com.tailscale.tailscaled.plist
sudo launchctl enable system/com.tailscale.tailscaled
sudo launchctl bootstrap system /Library/LaunchDaemons/com.tailscale.tailscaled.plist
```

```bash
sudo brew services start tailscale
sudo tailscale up
sudo tailscale status
```


## Git setup

(This changes in case CICD is setup )

Install `gh` for authenticting with Github:  
```bash
brew install gh
gh auth login
gh auth setup-git
```

Fork `ai-in-the-box` private repository in your Github. 

Clone your forked private repo to `/idata`:  
```bash
git clone https://github.com/duretech/ai-in-the-box.git
```


## Database

We will install postgres-18 for our work:  

1. Install PostgreSQL 18 (client + server, matching what's installed now)
```cmd
brew install postgresql@18
```

2. Link binaries onto PATH (Homebrew keg-only formulas need this)
```cmd
echo 'export PATH="/opt/homebrew/opt/postgresql@18/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

3. Start as a background service (auto-starts on login)
```cmd
brew services start postgresql@18
```

4. Verify
```cmd
psql --version
pg_dump --version
psql -d postgres -c "SHOW server_version;"
```

5. Extras:
We need vector embedding store
```cmd
brew install pgvector
```

We will now backup and restore the UAT version of database from main server to macmini:  
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
    rsync -avz /path/to/uat_db_name.dump macmini_user@macmini_hostname:/path/to/user/home/
    ```
4. Then on macmini:
    ```bash
    sudo -u postgres psql intellicare_uat < uat_db_name.dump
    ```


## Storage

### Project Mount Point

Create a new project dir `idata` owned by root:  
```bash
sudo mkdir /System/Volumes/Data/idata
``` 


### HF and UV Cache

In `~/.zshrc` add following lines:  
```bash
export HF_HOME=/System/Volumes/Data/idata/.cache/huggingface
export UV_CACHE_DIR=/System/Volumes/Data/idata/.cache/uv
```
Refresh your terminal : `source ~/.zshrc`

Create home and cache dirs in `/idata` for HF and UV:  
```bash
sudo mkdir -p $HF_HOME
sudo mkdir -p $UV_CACHE_DIR
```

Change ownership of these root owned caches:
```bash
sudo chown -R openclaw:wheel .cache
```

## Network

### Nginx

Download and install nginx:  
```bash
brew install nginx
brew services start nginx
```

## Tools


### dust

Convenient `du` wrapper in rust:  
```bash
curl -sSfL https://raw.githubusercontent.com/bootandy/dust/refs/heads/master/install.sh | sh
```

### fzf

Fuzzy finder for dev productivity:
```bash
brew install fzf
```

For key-bindings to work, perform shell integration by adding this line to `~/.zshrc`:
```bash
source <(fzf --zsh)
```

### btop

Resource monitor:  
```bash
brew install btop
```

### key-movements

If you like linux stype key movements on mac, add following to `~/.zshrc`:  
```bash
# Bind Ctrl + Left Arrow / Alt + Left Arrow to move back a word
bindkey "^[[1;5D" backward-word
bindkey "^[[1;3D" backward-word

# Bind Ctrl + Right Arrow / Alt + Right Arrow to move forward a word
bindkey "^[[1;5C" forward-word
bindkey "^[[1;3C" forward-word
autoload -U select-word-style
select-word-style bash
```