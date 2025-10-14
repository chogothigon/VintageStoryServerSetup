## Introduction
### What is Vintage Story
 
 Vintage Story is a voxel-based sandbox survival game emphasizing exploration, crafting, and creativity. Developed and published by Anego Studios, Vintage Story was originally created as a Minecraft mod called Vintagecraft to fill the hardcore survivalist niche. However, under the guidance of Tryon and Irena Madlener, the project has grown over the years into a sophisticated and increasingly popular indie title that stands on its own. 

As one of my favorite video games I have always looked for ways to play Vintage Story with my friends. Unfortunately, the options for multiplayer servers aren't very convenient, and although there are some community guides scattered across forums, many are outdated or assume a high level of prior knowledge. 
 
By their nature, indie titles often experience rapid developmental changes which, while beneficial for gameplay purposes, create problems for mod and server support. As a result, even creating a simple server for you and your friends can feel daunting and frustrating as instructions lead to dead ends and new updates lead to errors. New players especially feel frustrated or stuck on preventable errors, ruining what should be a great experience by adding unnecessary friction. This guide hopes to help you avoid that by streamlining the entire process. Instead of needing to piece together information from multiple sources, this tutorial will help walk you through the entire process of creating and managing a Vintage Story server from start to finish.

### Client vs Server Hosting

While the game is fully playable in single-player mode, the true potential of Vintage Story shines in multiplayer worlds. Because of this, the developers have built in tools to help players directly host the game from their own client. However, while running the game server through your client can technically work, it tends to be unstable, still requires messy network configuration, leaves security concerns, and only runs when you're actively playing. 

If you are looking for a quick fix this might be the best path forward for you and you can find more information here at https://www.vintagestory.at/selfhosting/. However if you have the resources and a little time I would strongly encourage you to try setting up a dedicated server. 

Running a dedicated server allows players to share a persistent world, collaborate on massive builds, and engage in long-term survival experiences that go far beyond what a single-player save can offer. Hosting your own Vintage Story server provides a level of control and customization that you can't get from client side worlds. A dedicated server can decide which mods are installed, how the world is configured, and enable better moderation. Plus, you can create backups, design firewall permissions, and improve redundancy. That’s right, you’re not goofing off. You're just refining your network and server administration skills.
### Tutorial overview

This tutorial won't cover every possible command or feature for running a server, but it will provide you with a solid foundation to get started. For my example, I'm using a virtual machine running Ubuntu Server 24.04 inside a Proxmox hypervisor, but the instructions below will work on nearly any Linux machine. We'll begin with the initial installation, move into security design, then reliability considerations, and finally end with scripts and modding support. By the end, you'll have a stable and secure dedicated server ready for you and your friends to enjoy.
### Game Server Requirements
##### Software
OS: Linux
Dependencies: .NET Runtime 8.0
Linux Server Game File: https://account.vintagestory.at/downloads

##### Hardware
Memory: 1 GB + 300 MB per Player
CPU: 4 Threads recommended. Frequency: 1 GHz base + 150 MHz per player
## Installation

To get the server running will take some initial setup. We will be working through the terminal for most of this guide however you will need access to a web browser to get the link for the latest version of Vintage Story.
#### 1. System Preparation

##### Update your system

Start by updating your Linux distro by running this command:

```
sudo apt-get update && sudo apt-get upgrade -y
```

This will ensure that you won't run into bugs caused by outdated software.
##### Install .NET runtime 8

Next we will install .NET runtime. Although previous versions of Vintage Story used .NET version 7, Vintage Story version 1.21+ requires .NET version 8. Without the proper version the server will crash upon boot so be sure to check that this is set up correctly before proceeding.

Start by installing .NET version 8 with this command:

```
sudo apt install -y dotnet-sdk-8.0
```

You can check if you installed it successfully by running this command:

```
dotnet --version
```

If it was installed correctly you should see something like this `8.0.120`.

##### Create a user

Running the game server as the root user can lead to very serious security risks. To mitigate that we are going to run the server using a new user called `gameserver`, which will have limited permissions tied to only the files required to run the server.

This user can be created by running this command:

```
adduser --shell /bin/bash --disabled-password gameserver
```

Press enter to input default blank values for all of the following fields and then enter Y when prompted.
##### Create game directories

After creating the user we need to make the directories the server files will run from. Run these commands to build a server, data, and system directory:

```
mkdir -p /srv/gameserver/vintagestory
mkdir -p /srv/gameserver/data/vs
mkdir -p /usr/lib/systemd/system
```
##### Download and extract game files

Finally we will download the game files to the vintagestory directory. Change your directory using this command:

```
cd /srv/gameserver/vintagestory
```

Go to [http://account.vintagestory.at/downloads](http://account.vintagestory.at/downloads) and copy the link of the newest `Linux_tar.gz Archive (server only)` file found under the `Show all downloads and mirrors of Vintage Story` section.

Download the game server file using `wget` via the terminal (This example is for version 1.21.4 you will need to replace the link with the link you copied)

```
wget https://cdn.vintagestory.at/gamefiles/stable/vs_server_linux-x64_1.21.4.tar.gz
```

Extract the tar.gz package and change the owner to your `gameserver` user:

```
tar xzf vs_server_linux-x64_*.*.*.tar.gz
rm vs_server_linux-x64_*.*.*.tar.gz
chown -R gameserver:gameserver /srv/gameserver
```

Congratulations! Now you have all the parts set up and ready to start configuring your server.

#### 2. Server Configuration and Startup

After downloading all the required resources we can now startup the server but first we need to do some configuration.

##### Create systemd service unit file

Using this command create a service unit file which we will run the server from:

```
nano /usr/lib/systemd/system/vintagestoryserver.service
```

This will open up a text file in the terminal. Copy the following content into the newly created file and then press `ctrl-s` to save and then `ctrl-x` to exit.

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

##### Create config file for rsyslog

Next we will create a configuration file just like we did with the previous file using:

```
nano /etc/rsyslog.d/vintagestoryserver.conf
```

Again this will open up a text file in the terminal. Copy the following content into the file and then press `ctrl-s` to save and then `ctrl-x` to exit.

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

##### Restart rsyslog and enable server

To active rsystlog and start your vintagestory server type these commands:

```
systemctl restart rsyslog.service;
systemctl enable vintagestoryserver.service
systemctl start vintagestoryserver.service
systemctl status vintagestoryserver.service
```

Assuming everything is working your terminal should look something like this:
![[Working Server.png]]
(exit using `ctrl-c`)

If it does, congratulations! your server is now up and running. Unfortunately it's still a little too secure and due to the default whitelist and router configurations, nobody can access it just yet.
#### 3. Server Security and Management

Now that you have a functional server it's time to open it up to the world. Configuration from this point forward is more up to the individual server admin so feel free to change things if you want. For example, if you want to you can continue to use the whitelist for access control instead of disabling. More information on that can be found here https://wiki.vintagestory.at/Guide:Dedicated_Server#Dedicated_server_on_Linux. However, I prefer to disable the whitelist and instead use a password so it is easier to share with my friends.

##### Server Configuration

Start by opening up the server configuration file:

```
nano /srv/gameserver/data/vs/serverconfig.json
```

Find the `"Password": null` line and change the null value to what you want your server's password to be. Then near the bottom change the `"WhitelistMode": 0` value to a 1. Save by pressing `ctrl-s` then exit with `ctrl-x`. This will turn off whitelist restrictions and set a password requirement to join but the changes still need to be applied with the command:

```
systemctl restart vintagestoryserver.service
```

You should now be able to join your server locally using its private IP address.

##### Port configuration

Finally to open your server to the internet you need to enable port forwarding from your router. This can look a little different from router to router but the basics are the same. You will need your servers private IP address which can be found with this command:

```
ip a
```

Once you have the private IP *(likely a 192.168.1.\* number)* configure your router's port forwarding to send traffic to your server's private IP using port 42420 with the tcp and udp protocol enabled. This can be fairly difficult to figure out but here is a good resource if you get stuck https://www.noip.com/support/knowledgebase/general-port-forwarding-guide. Once this is setup friends from outside of your network can connect by using your public IP address.

## Conclusion

Assuming you have done everything correctly you should now have a fully operational server that you and your friends can play on that is both stable, secure, and efficient. However, there is far more that you can do with your server than I could cover in this short tutorial. Everything from specific player permission configurations, to auto pausing features, or DNS configuration there is a lot more to explore and I would encourage you to keep exploring for yourself. Just remember to make a backup first.
