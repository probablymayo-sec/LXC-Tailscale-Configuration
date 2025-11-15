<h1>LXC-Tailscale-Configuration</h1>

In the process of attempting to try to allow myself to access my homelab from school, I ended up just allowing myself to access a single container remotely instead of the Proxmox host. Documentation and youtube videos were very confusing and I probably ended up making this more complicated than it needed to be. 

This container allows me to access it from anywhere without direct exposure (port forwarding) and without the need for a VPN configuration or changing firewall rules. 

Theoretically, I could configure this container to act as a gateway, but this would pose 2 problems. 
1. It would expose my entire network (so if someone got access to my tailscale they can see way more than just my proxmox)
2. if the container is down, then I can’t connect to my proxmox host either. So if I shut off this vm on accident or it fails in any way, the connection is gone. 

<h2>Configuration</h2>

Tailscale acts as a network interface, so when tailscale (or any program like it) wants to create a _VPN Tunnel Device_, it opens the **/dev/net/tun** directory and then uses a special **IO CTL System Call** which registers a new networking device within the kernel. 
- <img width="518" height="105" alt="image" src="https://github.com/user-attachments/assets/b41468bb-e8c6-4427-84e1-b44e2aa45878" />

when looking at interfaces on my proxmox host, if i see a new interface **tun0**, that’s the kernel making a brand new virtual interface based on tailscale’s request 

First i needed to download an LXC container template to my preferred storage “local” since this is also the place where my VM ISO files are located 
- <img width="1478" height="336" alt="image" src="https://github.com/user-attachments/assets/a3f18a44-553e-4d96-90b9-a35b4a2a9234" />

Now I have the root filesystem of the container image downloaded locally onto my server, I can use an arbitrary (in my case at least) number create the container as a non-privileged container (“**Create CT**” button)  
- <img width="715" height="554" alt="image" src="https://github.com/user-attachments/assets/97c37840-7fb5-4778-9418-d93c69229eed" />

I didn’t check off "start after creation" because I need to modify a configuration file via the shell:
- ```vi /etc/pve/lxc/1000.conf``` 

**Shift + G** to go to edit the bottom of the file and then click **G** by itself to go into _insert mode_ at the bottom of the file 
- If you fuck up and delete something on accident like I did first time, press **Escape** button followed by **:qa!** Which quit without saving any changes you made to the conf file 

I added these 2 lines into the bottom of the file and then once I do that it should look like the picture below
- ```lxc.cgroup2.devices.allow: c 10:200 rwm```
- ```lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file```

****these 2 lines basically let an LXC container use the TUN network device I mentioned earlier, which is needed by Tailscale and VPNs like Wireguard****

- <img width="982" height="282" alt="image" src="https://github.com/user-attachments/assets/0bb18143-98f4-45ae-8f82-51910c7a693f" />

**Escape** button followed by **:wq** which quit and saved my changes to the config file 

Now the LXC container is configured to have permissions to create a DevNet Tun device 
- The DevNet Tun Device is a special file that acts as a bridge between user space programs and the kernels networking stack. It's kind of like a method of allowing the application to pretend to be a network interface  

I typed ```pct start 1000``` to boot up the tailscale container 

I logged into the container and then went to the [tailscale.com/download/linux](https://tailscale.com/download/linux) page to get the tailscale linux installation script 
- I had to install curl on the container first since it didn’t come pre-installed, I had to update the container first and then download curl 

At the time of download, here was the installation script 
```curl -fsSL https://tailscale.com/install.sh | sh```

After this is done I used ```tailscale up --ssh``` to add this node to the tailnet

I was given a website to authenticate which added my container to my telnet, allowing the unprivileged LXC to be able to create a privileged network tunnel device (this is needed in order for the container to run tailscale inside it)

After all this, I installed the GUI on my container with ```apt install gnome-core -y```

Then i enabled the **gdm3** service 
- ```systemctl enable --now gdm3```

To check if gdm3 is running i used 
- ```systemctl status gdm3```

I added a new user (set username and password, everything else was left blank or default)

I added the user to the Sudoers file 
- ```usermod -aG sudo admin```

I installed **XRDP** and enabled it on boot automatically 
- ```apt install xrdp -y```
- ```systemctl enable --now xrdp```

What did I just do? I enabled RDP via within the same network which isn’t really useful to me at all 

I am also able to SSH into that specific container only. 



