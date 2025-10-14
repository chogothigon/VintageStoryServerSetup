## Introduction
### What is Vintage Story
 
 Vintage Story is a voxel-based sandbox survival game emphasizing exploration, crafting, and creativity. Developed by Anego Studios, it began as a Minecraft mod called _Vintagecraft_, created to fill the hardcore survivalist niche. However, under the guidance of Tryon and Irena Madlener, it has evolved into a fully independent and increasingly popular indie title offering deep survival mechanics and incredible modding support. 

As one of my favorite video games, I’ve always looked for ways to play Vintage Story with my friends. Unfortunately, the options for multiplayer servers aren’t very convenient, and although there are some community guides scattered across forums, many are outdated or assume a high level of prior knowledge.

By their nature, indie titles often experience rapid developmental changes which, while beneficial for gameplay, can cause problems for mod and server support. As a result, even creating a simple server for you and your friends can feel daunting and frustrating as instructions lead to dead ends and new updates introduce errors. New players especially may feel stuck or frustrated by preventable issues, turning what should be a great experience into one filled with unnecessary friction. This guide aims to help you avoid that by streamlining the entire process. Instead of needing to piece together information from multiple sources, this tutorial will walk you through everything required to create and manage a Vintage Story server from start to finish.

### Client vs Server Hosting

While the game is fully playable in single-player mode, the true potential of Vintage Story shines in multiplayer worlds. Because of this, the developers have built tools to let players directly host the game from their own client. However, while running the server through your client can technically work, it tends to be unstable, still requires messy network configuration, raises security concerns, and only runs when you’re actively playing.

If you’re looking for a quick setup, this might be the best option for you. You can find more information on that here:  
[https://www.vintagestory.at/selfhosting/](https://www.vintagestory.at/selfhosting/).

However, if you have the resources and a little time, I strongly encourage you to set up a **dedicated server** instead.

Running a dedicated server allows players to share a persistent world, collaborate on large builds, and engage in long-term survival experiences that go far beyond what a single-player save can offer. Hosting your own Vintage Story server also provides a level of control and customization you can’t get from client-side worlds. You can decide which mods are installed, how the world is configured, and enable better moderation. Plus, you can create backups, design firewall permissions, and improve redundancy. That’s right, you’re not just playing around, you’re refining your network and server administration skills.
### Tutorial overview

This tutorial won’t cover every possible command or feature for running a server, but it will provide a solid foundation to get started.

For this example, I’m using a virtual machine running **Ubuntu Server 24.04** inside a **Proxmox** hypervisor, but the instructions below will work on nearly any Linux machine. We’ll begin with the initial installation, move into server configuration, and then finish with access controls and backups. By the end, you’ll have a stable and secure dedicated server ready for you and your friends to enjoy.
### Game Server Requirements
##### Software
OS: Linux
Dependencies: .NET Runtime 8.0
Server File: [Vintage Story Server Downloads](https://account.vintagestory.at/downloads)

##### Hardware
Memory: 1 GB + 300 MB per Player
CPU: 4 threads recommended (1 GHz base + 150 MHz per player)
## Installation

Getting the server running requires some initial setup. We’ll be working through the terminal for most of this guide, though you’ll need a web browser to download the latest version of Vintage Story.
### System Preparation

#### Update your system

Start by updating your Linux distro:

```
sudo apt-get update && sudo apt-get upgrade -y
```

This ensures you won’t run into bugs caused by outdated software.
#### Install .NET runtime 8

Next, we’ll install the .NET runtime. Although previous versions of Vintage Story used .NET 7, version 1.21+ requires .NET 8. **Without the correct version, the server will crash upon startup, so make sure this step is done correctly before proceeding!**

Install .NET 8:

```
sudo apt install -y dotnet-sdk-8.0
```

Verify the installation:

```
dotnet --version
```

If it installed correctly, you should see something like `8.0.120`.

#### Create a user

Running the game server as the root user can lead to serious security risks. To mitigate that, we’ll run the server under a limited user account called `gameserver`.

Create it with:

```
adduser --shell /bin/bash --disabled-password gameserver
```

Press Enter to input default values for the prompts and type `Y` when asked to confirm.
#### Create game directories

After creating the user, we need to make the directories where the server files will reside. Run these commands to create the server, data, and system directories:

_NOTE: Blocks of commands like these can be run separately in order or all at once_
```
mkdir -p /srv/gameserver/vintagestory
mkdir -p /srv/gameserver/data/vs
mkdir -p /usr/lib/systemd/system
```
#### Download and extract game files

Navigate to the target directory:

```
cd /srv/gameserver/vintagestory
```

Go to [http://account.vintagestory.at/downloads](http://account.vintagestory.at/downloads) and find the latest **“Linux_tar.gz Archive (server only)”** under “Show all downloads and mirrors of Vintage Story.”  
Instead of downloading it directly, **right-click the link and select “Copy link address”** to copy the download URL.

Then using the link and `wget`, download the game files (replace the example link with your version)

```
wget https://cdn.vintagestory.at/gamefiles/stable/vs_server_linux-x64_1.21.4.tar.gz
```

Extract the tar.gz package and set the owner to your `gameserver` user:

```
tar xzf vs_server_linux-x64_*.*.*.tar.gz
rm vs_server_linux-x64_*.*.*.tar.gz
chown -R gameserver:gameserver /srv/gameserver
```

Congratulations! You now have all the components ready for configuration.

### Server Configuration and Startup

With everything downloaded, it’s time to configure and launch your server.
#### Create systemd service unit file

Create a service unit file:

```
nano /usr/lib/systemd/system/vintagestoryserver.service
```

This will open up a text file in the terminal. Paste the following content:

```
[Unit]
Description=Vintage Story Server Unit
After=network.target

[Service]
WorkingDirectory=/srv/gameserver/vintagestory
ExecStart=dotnet VintagestoryServer.dll --dataPath /srv/gameserver/data/vs
Restart=always
RestartSec=30
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=VSSRV
User=gameserver
Group=gameserver

[Install]
WantedBy=multi-user.target
```

The result should look like this:
![[Nano.png]]

 Save (`Ctrl+S`) and exit (`Ctrl+X`) the nano screen.
#### Create config file for rsyslog

To manage server logs more efficiently, create an rsyslog configuration file:

```
nano /etc/rsyslog.d/vintagestoryserver.conf
```

Paste the following content, then save and exit:

```
if $programname == 'VSSRV' then {
    if($msg contains "Chat") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/chat.log")
    } else if($msg contains "verließ") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/login.log")
    } else if($msg contains "join") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/login.log")
    } else if($msg contains "Server Warning") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/warn.log")
    } else if($msg contains "Server Notification") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/info.log")
    } else if($msg contains "Server Debug") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/debug.log")
    } else if($msg contains "Server Event") then {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/event.log")
    } else {
        action(type="omfile" dirCreateMode="0755" FileCreateMode="0644" File="/var/log/vintagestory-server/other.log")
    }
}
# Discard 
if $programname == 'VSSRV' then ~
```

#### Restart rsyslog and enable server

Activate rsyslog and start the Vintage Story server with:

```
systemctl restart rsyslog.service;
systemctl enable vintagestoryserver.service
systemctl start vintagestoryserver.service
systemctl status vintagestoryserver.service
```

If everything is working, your terminal should show the server as active:
![[Working Server.png]]

If everything is working, your terminal should show the server as active.  
(Exit with `Ctrl+C`.)
### Server Security and Management

Now that you have a functional server, it’s time to open it up for your friends to join. Configuration from here is mostly up to your preferences.

If you prefer to use the whitelist system, you can keep it enabled. Details are available here:  
[https://wiki.vintagestory.at/Guide:Dedicated_Server#Dedicated_server_on_Linux](https://wiki.vintagestory.at/Guide:Dedicated_Server#Dedicated_server_on_Linux).

However, I prefer to disable the whitelist and use a password instead, since it’s easier to share with friends.

#### Server Configuration

Open the server configuration file:

```
nano /srv/gameserver/data/vs/serverconfig.json
```

Find the `"Password": null` line and set it to your desired password. 
Then, near the bottom, change `"WhitelistMode": 0` to `1`.  
Save (`Ctrl+S`) and exit (`Ctrl+X`).

Apply your changes:

```
systemctl restart vintagestoryserver.service
```

You can now join your server locally using its private IP address.

#### Port configuration

To make your server accessible online, enable port forwarding in your router settings.

You’ll need your server’s private IP, which you can find with:

```
ip -4 -o addr show scope global | awk '{print $4}' | cut -d/ -f1
```
(if this doesn't work use "ip a")

Once you have it (usually something like `192.168.1.x`), forward **port 42420** with both **TCP** and **UDP** protocols enabled.

If you need help, this guide is a great resource:  
[https://www.noip.com/support/knowledgebase/general-port-forwarding-guide](https://www.noip.com/support/knowledgebase/general-port-forwarding-guide)

Once set up, your friends can connect using your **public IP address**.
#### Backups

To protect your world from griefers, corruption, or accidents, it’s important to take regular backups. Since backups require restarting the server, this script will automatically handle that once a day in the early morning.

Create the backup script:

```
nano /usr/local/bin/vs-backup.sh
```

Paste the following content, then save and exit:

```
#!/bin/bash
SERVICE="vintagestory.service"
DATA_DIR="/srv/gameserver/data/vs/Saves"
BACKUP_DIR="/srv/gameserver/backups"
LONGTERM_DIR="$BACKUP_DIR/longterm"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
DATE=$(date +"%Y-%m-%d")
BACKUP_FILE="$BACKUP_DIR/vs_backup_$TIMESTAMP.tar.gz"
LONGTERM_FILE="$LONGTERM_DIR/vs_backup_$DATE.tar.gz"
MAX_BACKUPS=7

echo "[Backup] $(date): Starting backup"

mkdir -p "$BACKUP_DIR" "$LONGTERM_DIR"

# Compress world folder
tar -czf "$BACKUP_FILE" -C "$DATA_DIR" .

# Keep only 7 recent backups
cd "$BACKUP_DIR" || exit
ls -t vs_backup_*.tar.gz | tail -n +$((MAX_BACKUPS + 1)) | xargs -r rm --

# Daily archive
if [ ! -f "$LONGTERM_FILE" ]; then
    cp "$BACKUP_FILE" "$LONGTERM_FILE"
    echo "[Backup] Archived to long-term: $LONGTERM_FILE"
fi

echo "[Backup] Done."
```

Make it executable:

```
sudo chmod +x /usr/local/bin/vs-backup.sh
```

Set up the daily backup schedule for your `gameserver` user:

```
sudo crontab -u gameserver -e
```

Add the following lines:

```
# Stop server for backup at 4:30 AM
30 4 * * * systemctl stop vintagestory.service

# Run backup at 4:32 AM
32 4 * * * /usr/local/bin/vs-backup.sh >> /var/log/vs-backup.log 2>&1

# Restart server at 4:40 AM
40 4 * * * systemctl start vintagestory.service
```

This sets up an automated backup routine that stops your server daily at 4:30 AM, backs up your world, and restarts the server automatically.
## Conclusion

If you followed all the steps correctly, you now have a fully operational Vintage Story server that’s both stable and secure.

However, there’s much more you can do beyond this tutorial. Everything from advanced permissions and automated moderation tools to DNS configuration and mod management. There are thousands of features you can add to customize your server so it works just the way you want it. Just, remember to **make a backup first**!
