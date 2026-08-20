# Deployment

## Production

### Frontend

1. Create frontend as a service for lauchdaemons:  
    ```bash
    sudo vim /Library/LaunchDaemons/com.intellicare.frontend.plist
    ```

    2. Add following to the `plist`:  

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">

    <plist version="1.0">
    <dict>

        <key>Label</key>
        <string>com.intellicare.frontend</string>

        <key>ProgramArguments</key>
        <array>
            <string>/Users/openclaw/.npm-global/bin/serve</string>
            <string>-s</string>
            <string>dist</string>
            <string>-l</string>
            <string>tcp://0.0.0.0:3002</string>
        </array>

        <key>WorkingDirectory</key>
        <string>/System/Volumes/Data/idata/ai-in-the-box/frontend</string>

        <key>EnvironmentVariables</key>
        <dict>
            <key>PATH</key>
            <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/Users/openclaw/.npm-global/bin</string>
        </dict>

        <key>RunAtLoad</key>
        <true/>

        <key>KeepAlive</key>
        <true/>

        <key>ThrottleInterval</key>
        <integer>5</integer>

        <key>StandardOutPath</key>
        <string>/System/Volumes/Data/idata/logs/frontend.log</string>

        <key>StandardErrorPath</key>
        <string>/System/Volumes/Data/idata/logs/frontend-error.log</string>

    </dict>
    </plist>
    ```

2. Install into `/Library/LaunchDaemons`:  

    ```bash
    sudo chown root:wheel /Library/LaunchDaemons/com.intellicare.frontend.plist
    sudo chmod 644 /Library/LaunchDaemons/com.intellicare.frontend.plist
    ```

4. Create the log dir: `mkdir -p /System/Volumes/Data/idata/ai-in-the-box/logs`

5. Rebuild frontend dist:  
    ```bash
    cd /System/Volumes/Data/idata/ai-in-the-box/frontend
    npm install
    npm run build
    ```

6. Load and start frontend as a daemon service:  
    ```bash
    sudo launchctl bootstrap system /Library/LaunchDaemons/com.intellicare.frontend.plist
    sudo launchctl enable system/com.intellicare.frontend
    sudo launchctl kickstart -k system/com.intellicare.frontend
    ```

### Backend

1. Create the plist file as `/Library/LaunchDaemons/com.intellicare.frontend.plist`:

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">

    <plist version="1.0">
    <dict>

        <key>Label</key>
        <string>com.intellicare.backend</string>

        <key>UserName</key>
        <string>openclaw</string>

        <key>GroupName</key>
        <string>staff</string>

        <key>ProgramArguments</key>
        <array>
            <string>/System/Volumes/Data/idata/ai-in-the-box/backend/.venv/bin/uvicorn</string>
            <string>app.main:app</string>
            <string>--host</string>
            <string>0.0.0.0</string>
            <string>--port</string>
            <string>8005</string>
            <string>--ws</string>
            <string>websockets-sansio</string>
        </array>

        <key>WorkingDirectory</key>
        <string>/System/Volumes/Data/idata/ai-in-the-box/backend</string>

        <key>EnvironmentVariables</key>
        <dict>
            <key>PYTHONUNBUFFERED</key>
            <string>1</string>
            <key>PATH</key>
            <string>/System/Volumes/Data/idata/ai-in-the-box/backend/.venv/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        </dict>

        <key>RunAtLoad</key>
        <true/>

        <key>KeepAlive</key>
        <true/>

        <key>ThrottleInterval</key>
        <integer>5</integer>

        <key>StandardOutPath</key>
        <string>/System/Volumes/Data/idata/logs/backend.log</string>

        <key>StandardErrorPath</key>
        <string>/System/Volumes/Data/idata/logs/backend-error.log</string>

    </dict>
    </plist>
    ```

2. Install into `/Library/LaunchDaemons`:  

    ```bash
    sudo chown root:wheel /Library/LaunchDaemons/com.intellicare.backend.plist
    sudo chmod 644 /Library/LaunchDaemons/com.intellicare.backend.plist
    ```

4. Create the log dir: `mkdir -p /System/Volumes/Data/idata/ai-in-the-box/logs`

5. Load and start it

    ```bash
    sudo launchctl bootstrap system /Library/LaunchDaemons/com.intellicare.backend.plist
    sudo launchctl enable system/com.intellicare.backend
    sudo launchctl kickstart -k system/com.intellicare.backend
    ```

6. Verify

    ```bash
    sudo launchctl print system/com.intellicare.backend | head -20
    curl -s http://127.0.0.1:8005/docs -o /dev/null -w "%{http_code}\n"
    tail -f /System/Volumes/Data/idata/logs/backend.log /System/Volumes/Data/idata/logs/backend-error.log
    ```

## Nginx


Add configurations for intellicare app:
```bash
sudo vim opt/homebrew/etc/nginx/servers/intellicare.conf
```

```bash
server {
    listen 3000;
    server_name _;

    location ~ ^/(api|analytics|sessions|dhis2-sync|dhis2-configs|field-corrections|system-config|config|forms|asr) {
        proxy_pass http://127.0.0.1:8005;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /ws {
        proxy_pass http://127.0.0.1:8005;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    location / {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Validate the `.conf` file:
```bash
sudo /opt/homebrew/bin/nginx -t -c /opt/homebrew/etc/nginx/servers/intellicare.conf
```

Gracefull reload without dropping any connections:  
```bash
sudo launchctl kickstart -k system/homebrew.mxcl.nginx
```

Else force restart:  
```bash
sudo brew services restart nginx
```

Verify it is running:  
```bash
sudo launchctl print system/homebrew.mxcl.nginx | head -n 40
lsof -nP -iTCP -sTCP:LISTEN | grep nginx
```

## Postgres

We launch Postgres with LauchDaemons too instead of LauchAgents to run on power-cuts.  
Postgres likes to run as non-root, however. So we need to edit the lauchdaemons plist:  

```bash
sudo cp /opt/homebrew/opt/postgresql@18/homebrew.mxcl.postgresql@18.plist /Library/LaunchDaemons/homebrew.mxcl.postgresql@18.plist 
sudo /usr/libexec/PlistBuddy -c "Add :UserName string openclaw" /Library/LaunchDaemons/homebrew.mxcl.postgresql@18.plist
sudo chown root:wheel /Library/LaunchDaemons/homebrew.mxcl.postgresql@18.plist
sudo chown root:wheel /Library/LaunchDaemons/homebrew.mxcl.postgresql@18.plist
```

Now reload and launch:  
```bash
sudo launchctl bootout system/homebrew.mxcl.postgresql@18 2>/dev/null || true
sudo launchctl enable system/homebrew.mxcl.postgresql@18
sudo launchctl bootstrap system homebrew.mxcl.postgresql@18.plist
sudo launchctl kickstart -k system/homebrew.mxcl.postgresql@18
```

Verify that it works:  
```bash
sudo launchctl print system/homebrew.mxcl.postgresql@18 | head -n 60
lsof -nP -iTCP:5432 -sTCP:LISTEN
pg_isready -h 127.0.0.1 -p 5432
```

## Tailscale

We use Tailscale funnel to create a public tunnel:  
```bash
sudo tailscale set --operator=<mac_username>
```

This will a public url which does not rotate, something like `https://dures-mac-mini.tail6b69fd.ts.net/`. Use this url for UAT purposes.

Go to Tailscale web console and change the name of the funnel to your liking, eg. `intellicare-edge-hospital.tail6b69fd.ts.net`.

```bash
tailscale funnel reset
tailscale funnel --bg 3000
```