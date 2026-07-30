# Optimizations

## Diable Auto-updates of 3rd party softwares

Disable:
```bash
sudo launchctl bootout system/com.microsoft.autoupdate.helper 2>/dev/null || true
sudo launchctl disable system/com.microsoft.autoupdate.helper
```

In case you want to re-enable them:  
```bash
sudo launchctl enable system/com.microsoft.autoupdate.helper
sudo launchctl bootstrap system /Library/LaunchDaemons/com.microsoft.autoupdate.helper.plist
```