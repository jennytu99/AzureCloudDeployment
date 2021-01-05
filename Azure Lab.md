# Secure Cloud Network Deployment Guide

###### This cumulative step by step guide is put together by myself, however the teaching credit goes to Trilogy 2U. This guide will be creating a secure cloud environment through the methods of Infrastructure as a Service (IaaS) in Azure. It will be using Jump-box as a provisioner, creating Virtual Machines serving as DVWA webservers. This guide will be using the free tier in Azure. 

__Infrastructure as a Structure, IaaS__ offers pay-as-you-go access to:

+ storage
+ networking
+ servers
+ other computing resources in the cloud

The benefits include affordability, high availability and guarantees that base machines are up-to-date and patched at the end of deployment. This means ease of of access to security controls through provider-enforced basic access management.

---

##### Make sure you sign up for an Azure account prior to starting this walkthrough.

---

#### Over view of what we will be creating the in following order:

1. Resource Group
2. Virtual Network
3. Network Security Group
4. Virtual Machine setup
5. JumpBox as Provsioner
    - Installing Docker, Ansible

### 1. Creating a Resource Group
*Resource groups* are the central area where multiple instances can be grouped together to allow engineers to sort related resources into particular groups, each can be easily located by name.

In this example, we will be creating a resource group that will:
 - Host a Jumpbox with Ansible container
 - 3 DVWA Containers, Network Security Groups
 - Virtual Networks and load balancer.


        In the Azure Portal:
            - search 'resource group' in the search bar
            - click the `+ Add` button
        Under the Basics tab:
            - name it "RedTeamRG"
            - use the default region, this will show automatically depending on your location. E.g West-US2
            - click review + create

### 2. Creating a Virtual Network
A *virtual network* consists of a collection of virtual machines that communicate with each other. These virtual machines that are on the same virtual networks can be in completely different data centers, but perform as if they are wired. Thus provide improved availability. It is also much easily reconfigured by using the Azure portal instead of rewiring physical networks to improve segmentation, which also results in less human error.

Just like physical machines, a virtual network in Azure has:
- vNICs (virtual Network Interface Cards):
- IP addresses
- Subnets
    
        - search 'virtual network'
        - click the + Add button
        - name it "RTVNet"
        - select the resource group that was created previously 'RedTeamRG'
        - select the region closest to your locaiton. E.g WEST-US
  
Under the IP address tab:
    We are going to leave subnets as *default* when creating the virtual network.
        
        10.0.0.0/16 for the large network
        10.0.0.0/24 for the first subnet

    Leave the security and tag tabs as default, no configurations there.

        Click review + create

We do not need to configure a router for DHCP, as these are configured automatically. We just need to define the boundaries of the network and Azure does the rest.

## 3. Creating a Network Security Group

We will set up this *network security group* which will block & allow traffic to the virtual network and between machines on that network.

    - search 'network security group'
    - click `+ Add` 
    - name the group 'NSG-RT'
    - once created, go to resource
    - click the inbound rules on the left panel

### Creating NSG rules

We will create a rule that allows `RDP on port 3389`

*Source* is the source computer/machine. It can be a single IP, a range of addresses, an application security group or a service tag.
    
    - choose 'IP Address' from the dropdown and paste in your external IPv4 address

*Source Port Ranges* are generated randomly, set it as wild card (*) to signify all ports.

*Destination* for now, choose VirtualNetwork

    'DESTINATION PORT RANGES' RDP uses port 3389

    'PROTOCOL' Choose Any to specific TCP, UDP, ICMP

    'ACTION' Priority number allows rules to be read and sorted based on their priority number, low to high. 100 would be low if there were any number greater than 100, therefore 100 would be read first.

    'NAME/DESCRIPTION' Name it "Allow-RDP", with a description: allow RDP from external IP over port 3389

## 4. Creating a Virtual Machine and adding it to your network

#### Creating the first Virtual Machine, VM 1 - This wlil be the Jump-Box container
Located under the basics tab

    - search for virtual machines
    - click the `+ Add` button
    
    Select:

        'RESOURCE GROUP':  RedTeamRG

        'NAME' "Jump Box Provisioner"   
        
    by default Azure should populate a VM and Region
 
        'REGION' same region, WEST-US2

        'AVAILABILITY OPTIONS' we won't need redundancy for the Jump-Box. We will only have one administrator using Jump-Box at a time. 

        'IMAGE' Ubuntu Server 18.04

        'VM' that has: Standard - B1s, 1 CPU, 1 RAM
        //if the machine is not available, try changing regions

#### Once the Jump-Box container is created, we need to access it via SSH. However, instead of using a password we will practice secure system setup by generating a SSH key pair.

Under the Administrator Account section, Basics page for VM-1 in Azure:

    select 'Use existing public key' from the drop down

On your local machine, open a terminal and run 
    
    ssh-keygen

Save the SSH key into the default prompted directory 
    
    ~/.ssh/id_rsa

Do not enter a password when prompted.

    cat ~/.ssh/id_rsa.pub

Copy and paste the SSH key string into the Adminstrator Account section on Basics page for VM-1.

    'AUTHENTICATION TYPE' SSH public key

    'USERNAME' any username that you can remember. e.g: sysadmin

    'SSH PUBLIC KEY' 'use existing public key' paste the public key string 
        paste the SSH key string

    'PUBLIC INBOUND PORTS & SELECT INBOUND PORTS' ignore this setting, it will be overwritten by the NSG

Completing these steps will allow for SSH access through the newly created administrator account.

#### On to setting Networking tab

    'VIRTUAL NETWORK' choose the VNet you created for the RedTeam

    'SUBNET' choose the subnet that was created earlier

    'PUBLIC IP' Create New -> Static, Give a unique name.

    'NIC Network Security Group' choose the advanced option to specify our custom security group
    
    `Accelerated networking` Keep as the default setting (Off).

    'CONFIGURE NETWORK SECURITY GROUP' choose the Red Team NSG

    'ACCELERATED NETWORKING' keep as default, off

    'LOAD BALANCING' keep as default, no

    REVIEW + CREATE

## VM's 2, 3, 4 the WEB VMs, DVWA

- Name each "Web-1, Web-2, Web-3"
- VMs should be in the same resource group used for all other resources
- VMs should be located in the same region as RG & NSG
- admin username should be the same for all 3 machines
- create a new SSh key for remote connections
- choose the VM option that has `standard - B1ms, 1 CPU, 2 RAM`
- ensure that all VMs are in the same availability set. Under availability options, select availability set, click 'create new'. Name it: `RedTeamAS` After creating it for the first VM, select for the 2,3.

In the Networking tab:    

     - `Virtual network` choose the VNet you created, RTVNet
     - `Subnet` choose the subnet that was created earlier
     - `Public IP` NONE. make sure these don't have public IPs
     - `NIC network security group` choose advanced option to specify NSG, RedTeamNSG
     - `Accelerated networking` default, off
     - `Load balancing` default, no

These VMs will not be accessible because we have not adjusted the security group rule. Currently it is blocking all traffic.

## JumpBox as Provisioner

We will be using Jump-Box as a Provisioner. Specifically vagrant. Provisioners are tools that automatically configure VMs or containers for you. Instead of logging into a machine and issuing commands like `apt get` or editing configuration files yourself, you can use a provsioner to do this automatically. They drastically reduce the potential for human error and make it easy to configure potentially thousands of identical machines all at once.

#### A jump box is essentially identical to a gateway router. We will be exposing it to the public internet thru SSH port 22. The jump box will sit infront of other machines that are not exposed to the public internet. It controls access to the othe rmachines by allowing connections from specific IP addresses and forwarding to those machines. This pattern is known as "Fanning in"

### Jump Box Administration
To connect to our Web VM's, we can create a security group rule to allow SSH connections only from your current IP address.

    - find your own IP address
    - go to Azure and click security group listed under resource group
    - create rule allowing SSH connections from your IP address
    - select Inbound security rules

        `Source` your IP address
        `Source port ranges` Any or *
        `Destination` VirtualNetwork or better, specify the internal IP of jump-box container to limit traffic.
        `Destination port ranges` port 22
        `Protocol` Any ot TCP
        `Priority` lower number than your previous rule that denied all traffic.
        `Name` Give it a name, e.g: SSH
        `Description` e.g: Allow SSH from my IP

From your commandline on your local machine, you can SSH to the VM for administration.
    e.g: `ssh sysadmin@VM-publicIP`

You can check sudo permissions just in case: `sudo -l`

Options: You could set a static IP address to avoid creating multiple rules to allow access.

-------

`Infrastructure as code (IaC)` is the idea that the configuratios for all of the VMs, containers, and networks in your deployment should be defined in text files, which can be used with provisioners to automatically recreate machines and networks whenever necessary. Pirmary benefit to IaC is that everyone can see exactly how the network is configured by reading text files. These can be version controlled in a tool like Git, Apple Time Machine, and Microsoft OneDrive.

`Continuous Integration / Continuous Deployment (CI/CD)`
is the concept of automatically updating machines on your network whenever your IaC files change. In other words, whenever you change a machine's configuration file, `CI` ensures that a new version of that machine is built immediately. `CD` ensures that this new version is automatically deployed to your live environment. The primary advantage of CI/CD is that it allows you to manage your entire network by simply updating IaC text files.

As of now, none of our VMs are accessible, this is an example of 'secure configuration' as opposed to 'secure architecture'

`Secure configuration` ensures that an individual VM or network is protected from intrustion using well-considered rules, such as access control policies and firewall rules. A securely configured VM or network is secure because it follows the right rules.

`Secure architecture` ensures that a poorly configured or malfunctioning individual machine can only cause a limited amount of damage. A secure netwrok is secure because it is "structurally sound". It deters and contains the effects of a breach. Ensures that a damage to a single machine doesn't take down the entire network.

Secure configuration supports and protects a secure architecture.


-----

## Install Docker on Jump-Box VM and use Docker to install Ansible, the provisioning tool we'll be using to connect to web VM's

First, install Docker:

    1. `ssh sysadmin@jump-box-ip`
    2. `apt-get update` or `apt-get upgrade`
    3. `sudo apt install docker.io` This installs Docker image
    4. `sudo systemctl status docker` This checks the status of Docker
        If Docker is not running, start it with `sudo systemctl start docker`

Use Docker to install Ansible:

    1. `sudo docker pull cyberxsecurity/cyberxsecurity/ansible`
    2. `sudo docker run -ti cyberxsecurity/ansible:latest bash`

        -`ti` stands for terminal and interactive. It sets up the container to allow us to run a terminal and interact with the container.
        bash is the command we will be running inside the conatiner, this will give us a shell to control the container
    `exit` to quit

Create a new security group that allows the Jump-box machine to have full access to the VNet.

    - Locate the private IP address of the Jump-box by going

    Go to your security group settings and create an inbound rule. Create rules allowing SSH connections from your IP address.

    `Source` Use the IP Addresses setting with your jump box's internal IP address in the field.

    `Source port ranges` Any or * can be listed here.

    `Destination` Set to VirtualNetwork.

    `Destination port ranges` Only allow SSH, 22.

    `Protocol` Set to Any or TCP.

    `Action` Set to Allow traffic from your jump box.

    `Priority` Priority must be a lower number than your rule to deny all traffic.

    `Name` Name this rule anything you like, but it should describe the rule. For example: SSH from Jump Box.

    `Description` Write a short description similar to: "Allow SSH from the jump box IP."


### On the Jump-Box VM, run Ansible to configure other servers

In your local machine:

    1. `ssh admin@jump-box-ip`
    2. `docker run` this should only be ran on the first time. Or else we'd be creating multiple containers
    3. `docker container list -a` lists all containers created on the system
    4. `sudo docker start name_of_container`
    5. `sudo docker ps` lists all running containers
    6. `sudo docker attach container_name`

The prompt should change to something like `root@23b86e1d62ad:~#`

We will need to create a new SSH key pair on the Ansible container and reset the SSH keys on our Web VMs to ues the SSH id_rsa.pub file from our Ansible container.

    `ssh-keygen`
    `cat ~/.ssh/id_rsa.pub` w/ no password. 
Go back to Azure portal and and paste the SSHkey into the new WEB-VM, with mode selected to `reset ssh public key`. Then copy and paste the new ssh key.

    ssh <username>@<webVM-IP>: accept the key

    exit : close the ssh connection to return to the ansible container.

run  `ansible` to make changes to the configuration files to let Ansible make connections:

1. Ansible needs to know which administrative username it should use when making SSH connections. This will be the admin username you used when you created your Web-VM's.

2. Ansible needs to know the IP address of the VM you want it to connect to.

Each of these settings is located in a different file, but all Ansible configurations live in:
    
    /etc/ansible


Run `cd /etc/ansible` and then `ls` to show all the files

    `ansible.cfg`: The file with the setting for the admin name that should be used.

    `hosts`: The file with all of the IP addresses that should be used.

To ensure that the user matches the admin username we use when we create the new VM we have to edit the *remote_user* section in *ansible.cfg*

    # default user to use for playbooks if user is not specified
    # (/usr/bin/ansible will use current user as default)

    remote_user = Your_User_Name

    exit

*hosts* file must contain IP addresses for any machines that Ansible connects to.

    nano hosts

Machines can be grouped together under headers using brackets:

[webservers] or 
[databases] or 
[workstations]

Headers can all hold different groups of IP addresses, which Ansible can run configurations on individually or together.

For now, we will only have one web server, so we can add our IP to the provided web server header.

Uncomment the [webservers] header line.

    Add the IP address of the webserver under the header [webservers] along with the python line:

    ansible_python_interpreter=/usr/bin/python3

    exit

To verfiy the connection, ping from ansible.

    ansible webservers -m ping
    or
    ansible all -m ping

Output of a successful ping will result with *ping:pong*


## Writing Ansible playbooks to configure VMs

Connect to your jump box and then into your Ansible container

    -  run `ssh admin@jump.box.ip`
    - `sudo docker start container_name` to start your container
    - `sudo docker container list -a` if you want to see your container's name
    - `sudo docker ps` to view running containers
    - `sudo docker attach container_name` to get a shell on your Ansible container
    - `cd /etc/ansible` and `ls` to see the files 
    - `nano my-playbook.yml` to create a YAML file

This is what a basic apache ansible playbook looks like:

   ---
      - name: My first playbook
        hosts: webservers
        become: true
        tasks:

        - name: Install apache httpd (state=present is optional)
          apt:
            name: apache2
            state: present

Save and close the playbook.
Run `ansible-playbook my-playbook.yml`
    there should be a detailed output of each task that Ansible has completed. Tells us the name of playbook and name of tasks created.

Ansible modules such as [apt, pip, docker-container] can be used

## Load Balancing
A way to mitigate DoS attacks is to have multiple servers running the same website with a load balancer in front of them. It provides a website an external IP address that is accessed by the internet. It recevies any traffic that comes into the website and distributes it across multiple servers. It also has a health probe function that checks regularly to make sure all of the machines behind the load balancer are functioning before sending traffic to them. 
- go to Azure portal and search load balancer
- click the + Add button
- select the relevant resource group
- provide a name `Red-Team-LB`
- select the same region
- type: public
- SKU: basic
- select the same resource group as your other resources
- select create new for the public IP address setting
- give the public ip a name `Red-Team-LB`
- choose static for IP assignment
- click on review + create and confirm

Configure load balancer and security group to work together expose port 80 of the VM to the inernet

#### Add a health probe
- Name: RedteamProbe
- Protocol: TCP
- Port: 80
- Interval: 5
- Unhealthy threshold: 2

#### Create a backend pool and add your VM to it
- Name: RedTeamPool
- Virtualnewtowkr: RedNet
- IP version: IPv4
- Associated to: Virtual machine
- Virtual Machine: DVWA-VM1, IP address: ipconfig1(10.0.0.6)

Now that we have a load balancer running, we want to make sure it is configured properly to allow traffic to the VM backend pool.
    - the security group will need to be configured to allow web traffic into the VNet from the load balancer.

- Open Azure, click on `load balancing rules` on left panel
- click the + add button
- Choose the backend pool and health probe we created

### Security Configuration

1. Create a load balancing rule to forward port 80 from the load balancer to your Red Team VNet.

- Name: Give the rule an appropriate name that you will recognize later.

- IP Version: This should stay on IPv4.

- Frontend IP address: There should only be one option here.

- Protocol: Protocol is TCP for standard website traffic.

- Port: Port is 80.

- Backend port: Backend port is also 80.

- Backend pool and Health probe: Select your backend pool and your health probe.

- Session persistence: This should be changed to Client IP and protocol.<Remember, these servers will be used by the Red Team to practice attacking machines. If the session changes to another server in the middle of their attack, it could stop them from successfully completing their training.

- Idle timeout: This can remain the default (4 minutes).

- Floating IP: This can remain the default (Disabled).

2. Create a new security group rule to allow port 80 traffic from the internet to your internal VNet.

- Source: Change this your external IPv4 address.

- Source port ranges: We want to allow Any source port, because they are chosen at random by the source computer.

- Destination: We want the traffic to reach our VirtualNetwork.

- Destination port ranges: We only want to allow port 80.

- Protocol: Set the standard web protocol of TCP or Any.

- Action: Set to Allow traffic.

- Name: Choose an appropriate name that you can recognize later.

3. Remove the security group rule that blocks all traffic on your vnet to allow traffic from your load balancer through.

Remember that when we created this rule we were blocking traffic from the allow rules that were already in place. One of those rules allows traffic from load balancers.

Removing your default deny all rule will allow traffic through.

4. Verify that you can reach the DVWA app from your browser over the internet.

Open a web browser and enter the front-end IP address for your load balancer with /setup.php added to the IP address.
    For example: http://40.122.71.120/setup.php

<Note: With the stated configuration, you will not be able to access these machines from another location unless the security Group rule is changed.>

