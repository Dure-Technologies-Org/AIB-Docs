# Setup

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

3. Initialize a data directory (if not already present)
```cmd
initdb --locale=en_US.UTF-8 -D /opt/homebrew/var/postgresql@18
```

4. Start as a background service (auto-starts on login)
```cmd
brew services start postgresql@18
```

5. Verify
```cmd
psql --version
pg_dump --version
psql -d postgres -c "SHOW server_version;"
```

6. Extras:
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