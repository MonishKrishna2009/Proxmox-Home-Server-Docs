This is a simplified guide to installing Portainer within a Proxmox LXC for homelab use based on my personal notes. Portainer is a handy tool to manage your Docker containers through a clean web interface. This guide assumes you have an functional Proxmox environment and knowledge on how to harden your environment outside of what is described here.

This guide does not cover the following topics (though I'd like to write about these some day):

Docker (and Portainer) concepts
Configuring your network for secure access to Portainer or other services
Reverse proxy or setting up signed SSL certificates
With all that out of the way, let's get started!

Prepare the LXC for Portainer
Download the CT Template
In Proxmox, navigate to your node's local storage (named local by default)
Click CT Templates
Select the Templates button. This will display the default LXC container templates available for download
Search for debian-11-standard
Select the result
Click Download
Debian 11 Template

Create the LXC
Right click on your desired Proxmox node and select Create CT
Provide a desired Hostname and a Password (for the root user). Optionally, you can load an SSH Key File. Be sure to save the password somewhere secure!
Click Next

General
From the dropdown, choose the Template we downloaded earlier
Click Next

Template
Now we need to decide how much disk space our Portainer and Docker Containers can use. What you choose depends on how much disk space you have available and what you're planning on running. The value your are setting here is the maximum, which allows our LXC and the data inside of it to grow until it reaches this amount. Increasing the disk size in the future is very easy to do. Shrinking it, however, can be a bit complicated and risky.
Once you've chosen your Disk size amount, click Next

Disks
Much like Disk size, your CPU Cores selection is based on your individual circumstances. It is also the maximum your LXC can consume.
Once you've chosen your Cores amount, click Next

CPU
Much like Disk size and CPU Cores, your Memory and Swap resource selection is based on your individual circumstances. Once again, it is the maximum your LXC can consume. (Seeing a pattern here? 😀)
Once you've chosen your Memory and Swap amounts, click Next

Memory
If your networking configuration for your Proxmox setup is fairly basic, you likely won't need or want to change much on the Network screen. In my case, my Proxmox server is on a dedicated "server" VLAN switch port and therefore all my Proxmox VMs and Services will also be on that same network if I leave these settings at their default. The only option I typically change is to enable DHCP for IPv4 because I use a DHCP server to manage address assignments.
Make any desired Network changes then click Next

Network
I typically leave DNS at their defaults because I have Proxmox and DHCP settings configured to point things towards my internal DNS server.
Make any desired DNS changes then click Next

DNS
Finally, review all your settings for your new LXC. You can check the Start after created checkbox which will start the LXC as soon as you click Finish

Confirm
Install Docker and Portainer
Log into the LXC
Now that you've got a running LXC, it's time to log into it and setup Docker and Portainer.

Note: This can be done via SSH, however it will require additional network configuration with the LXC (and possibly your homelab network) to allow you to access the SSH service remotely.

Select the LXC from your Proxmox node
Select the LXC's Console tab. You should be presented with a login screen. If your screen is blank, hit enter
Enter root at the prompt
Enter the password you chose when setting up the LXC (this can be pasted using Right Click -> Paste). Note that any text entered at the Password: prompt will not display. If you think you messed up, use CTRL+C to try again.
You should have a standard console prompt
Logging in to the LXC console
Update the System
I like to make sure all the default packages are up-to-date. This can be done by running...

apt update && apt upgrade -y
Install Docker
I won't breakdown every command below. I would highly recommend you research them further with the official Docker docs.

Install the Docker Repository
apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
apt update
apt-cache policy docker-ce
Install the Docker Service
Install the Docker service using the repository we just added

apt install -y docker-ce
Verify Docker is working

docker run hello-world
Docker will automatically download the hello-world image and run the test container. If everything is setup correctly, you should see the following:

Testing docker install

Setup a Docker User
Right now the only way we can run Docker commands is through our root user. Docker automatically created a group called docker which any user can be added to in order to run Docker commands. Let's create a new user named docker and add them to the docker group.

useradd -m -s /bin/bash docker -g docker
You can now login as our new user and verify it has the right permissions to command Docker.

su docker
docker run hello-world
Setting up and testing the docker user

Install Portainer
It may come to no surprise that Portainer runs out of a Docker container itself. This makes updating and management of Portainer as easy as any other Docker container. Like the commands for installing Docker, we won't spend time explaining each thing in-depth. Instead, I suggest reading the official Portainer CE Docs

To begin, we will create a new storage volume for Portainer to keep the important data in.

docker volume create portainer_data
And now we can run the command to download the latest portainer-ce (Community Edition) image and run it in a new container with our volume mapped to it.

docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
Starting the Portainer LXC

Access Portainer
At this point you should have the Portainer container running within your LXC in Proxmox.

Find your LXC's IP Address
If you did not specify a static IP address during the setup of your LXC, you will need to find your LXC's IP. You can do that by checking your DHCP server or running the following within the LXC...

ip a
Determining LXC IP address

The bridged interface to your homelab network is likely eth0 and contains an IP address within the subnet you expect.

Log In to the Web Interface
Once you know your LXC's IP, you should be able to open up a web browser on your local computer and visit https://<LXC_IP>:9443/

Certificate Warning
Without a valid SSL certificate you always get a certificate warning in your browser. For now, this is OK to ignore and proceed anyway (In most browsers, you need to click More Details to view the option to continue). You can solve this warning by using something like a reverse proxy for all your homelab apps that can handle SSL certificates for you.

Certificate Warning

Interface Timeout
If there was too much time between you starting Portainer and accessing the interface for the first time, you will get this message.

Portainer timeout

This is a nice security feature to prevent accidentally starting and forgetting an unconfigured Portainer service, leaving it potentially exposed on your network for someone else to find and "setup" for you.

To try again, jump back into your LXC and restart the container...

docker container restart portainer
Set Up Admin
You can now set up your admin account which will be used to do all your initial configuration inside the Portainer web interface. It is best practice to choose a name other than admin. You can optionally disable statistics collection and review the privacy policy surrounding that.

Set up Portainer Admin

You're done! You can now manage your containers inside of a Proxmox LXC with Portainer

Portainer Interface

Conclusion
I hope some of this was beneficial to you. If you find any issues or have questions, please reach out using the contact methods at the top of this website.

If you're looking for suggestions on where to go next...

Stop/Restart/Rebuild your Portainer container to verify your portainer_data volume persists and your configurations are not lost
Setup backups for your LXCs within Proxmox (and test them!)
Create a Watchtower container to automatically update your containers on a schedule.
Configure your LXC/Proxmox/network hardware to limit access to the Portainer service from only trusted clients or local networks
Review the hardening guide for Portainer here: https://www.portainer.io/blog/how-to-secure-your-portainer-installation
Review the hardening guide for Docker here: https://docs.docker.com/engine/security/
Setup a reverse proxy to handle ports and SSL certificates
Register with Portainer.io for a free business license and try setting up multiple Portainer nodes across multiple LXCs
Configure your LXC as a new template so you can quickly build more Portainer LXCs
