# Project-1---UCB-Cybersec
First project assigned for class.  Files and documentation necessary for a virtual network that contains an ELK server managing 3 web servers with a DVWA

**Automated ELK Stack Deployment**

 
**Abstract**

This purpose of this document is to describe the process used to configure a network which was completed by setting up an ELK Stack server that is monitoring 3 virtual machines.  These 3 machines are each running a web application known as a “Damn Vulnerable Web Application” (DVWA) which is commonly used for learning and testing.  They are publicly accessible through a load balancer with a front facing public IP address. The administration of these web severs is performed through one virtual machine set up as a jumpbox with strict access controls.  Finally, the ELK Stack server is publicly accessible for monitoring purposes.  Access control is managed by whitelisting only specific IP addresses, allowing only specific ports (22) and using public keys instead of passwords.

 
**Network Topology: Configure for high availability, strict access control and security management**

Our first step to create this network is to setup the Resource Group under which all of our virtual networks will live.  We named this resource group “RedTeamNet.”  Inside this resource group, we need to create the 2 virtual networks and peer them.  The first virtual network will contain the Firewall, JumpBox, the Load Balancer, and the 3 virtual machines containing the web application.  The second virtual network will contain the Firewall managing access to the ELK server and the virtual machine containing the ELK server.  The rest of this section explains this network diagram:

<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/Project1-NetworkDiagram.vpd.pdf>

**Virtual Networks**

The network containing the web servers is named “RedTeamNet” and is set with a private IP range of 10.0.0.0/16.  This was the default set by Azure and for the purposes of this exercise there was no need to change it.  If we were building a live network, we would look at the number of nodes necessary and allow for scalability to set the specific range.  The second network is named “ProjectvNet” and is set with the private IP range of 10.1.0.0/16.  Because both of these networks are in different zones, in order for them to be able to communicate, we need to create a peering through Azure.

**Firewalls** 

We are managing access control through Network Security Groups (NSG) which essentially serve as virtual firewalls.  We’ve set up an NSG per network.  The NSG for RedTeamNet is named RedTeamNSG and the one for ProjectvNet is named ELKserver-nsg.  The first thing we need to do is set a deny all rule for both NSGs, then we manage the whitelist individually.

**_RedTeamNSG_**

The configuration of RedTeamNSG is as below:
 
<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/RedTeamNSG.jpg>

Where my public IP is blacked out, 10.0.0.4 is the jumpbox, 104.210.36 is the load balancer, and VirtualNetwork is all devices on the Virtual Network.

    - Rule 100 allows access from my public IP to the jumpbox using SSH on port 22.  For further access control, we have it set to use a private key 
    - Rule 101 allows access from the jumpbox to the rest of the devices on the network, also using SSH and port 22 with private keys.
    - Rule 121 allows public access to the load balancer through http to our virtual network.
    - Rule 65000 is a default rule, which allows our vnet to communicate internally.
    - Rule 65001 is what allows our load balancer to monitor our web servers and forward traffic to our web servers.

**_ELKserver-nsg_**

The configuration of ELKserver-nsg is as below:

<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/ELKserver-nsg.jpg>
 
Where my public IP is blacked out, 10.0.0.0/16 is RedTeamNet, and 10.1.0.5 is the ELK server.

    - Rule 1000 allows the web servers from our RedTeamNet to send the logs to the ELK server.
    - Rule 1010 allows connection to Kibana on the ELK server using port 5601.

**Virtual Machines**

We have a total of 5 virtual machines (VMs) in the two networks.  Below is the breakdown for them:

<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/VirtualMachines.jpg>
    
**_JumpBox_**

This machine is the point of contact for all administrative access to the rest of the network, including the peered network.  It is configured on RedTeamNet with no availability set and a public IP address.

**_Web-1 to Web-3_**

These are our web servers that contain the DVWA.  They are configured on RedTeamNet with no public IP address and as members of the same availability set.  These are accessible for admin purposes by SSH-ing into them from the JumpBox with a private key.  Public access to them is managed from the load balancer.  Because of this, their public IP address shows up as the load balancer’s public IP address.

**_ElkServer_**

The ElkServer is configured on ProjectvNet and is a member of its own availability set.  Access to it for administratve purposes goes through the JumpBox on port 22 through SSH.  For public access, port 5601 is opened on Kibana.

**Load Balancer**

The load balancer ensures the availability of the web app.  The application is installed on all three servers which are monitored by the load balancer.  All public access to the machines is managed through the load balancer which also makes sure the app is always available, if at least one of the servers is up and running.


**Installation of the DVWA and Elk**

Because we’re installing multiple instances of the DVWA and because our Elk server is running on a virtual machine, we chose to use ansible to automate the installation.  Using Ansible to automate the installation allows us to:

    - quickly deploy new machinesindividually or as a batch process, if necessary
    - make updates to the deployed machines as a batch process
    - test deployments and make changes to the installation files prior to making them live.

Ansible will live on our gateway.  The first step we need to take is to install docker on the jumpbox.  We do this by running a *sudo apt install docker.io*.  Then we need to pull the ansible container with *sudo docker pull cyberxsecurity/ansible*.  Finally, we run the container with *sudo docker run -ti cyberxsecurity/ansible:latest bash*.
	
Now that ansible is installed on our jumpbox we have the ability to create ansible playbooks to deploy multiple VMs.  Before we can run the playbooks, we need to modify the /etc/ansible/ansible.cfg file with (see screenshot below)

*remote_user = 'your admin name for the vms'*

<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/ansible-config-modification.jpg>
 
We must also modify the /etc/ansible/hosts file with specific computer groups to use in the playbooks.  In our case we’ll use “webservers” for our DVWA installation and “elkservers” for our Elk installation. (see screenshot below)
	 
<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/ansible-hosts-modification.jpg>

Now that we have ansible configured with the host groups and the username for our admin account we can proceed with running the playbooks.  Following are the playbooks we used:

<https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Diagrams/playbooks-used.jpg>

While logged in to the ansible container, we need to run the following commands to run the playbooks.

*ansible-playbook 'playbook file'*

In our case, specifically these commands in this order:
- *ansible-playbook dvwa_config.yml*
- *ansible-playbook elk_config.yml*
- *ansible-playbook install-filebeat.yml*
- *ansible-playbook install-metricbeat.yml*

List of YML files:

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/dvwa_config.yml>

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/elk_config.yml>

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/install-filebeat.yml>

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/install-metricbeat.yml>

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/Files/Filebeat.yml>

    <https://github.com/gagavitz/Project-1---UCB-Cybersec/blob/master/Ansible/Files/Metricbeat.yml>
