# Setup

## SSH

### Server-side

In `~/.ssh/config` add jetson config

```cmd
Host jetson
        HostName <jetson IP>
        User <jetson login username>
```
where `<jetson IP>` is either public IP or IP to which server can communicate to.


### Local-side

Save your local public keys to server `~/.ssh/authorized_keys`:
```cmd
ssh-copy-id <server login username>@<server IP>
```
