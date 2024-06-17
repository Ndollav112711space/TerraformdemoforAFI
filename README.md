The Design and Infrastructure
The goal of this short project is to design and deploy a generic site to site vpn connection between two Azure VPN gateways through IPsec on Azure using terraform and then later testing for connectivity all while performing a deep packet inspection or packet capture. This will show how we can troubleshoot connectivity issues that could arise from networking or firewall blocks in production environment. All the resources involved are provisioned with terraform
I will put the terraform code in a GitHub repo that you can access and review later.

I have two vnets, one Aramark and the other named city. Each vnet has two subnets. One subnet dedicated for the vpn gateways, and the other subnet used for virtual machine connectivity. Lastly, we have a bidirectional ipsec connection between both sites

 

Implementing the design and deploying the resources to Azure with terraform.

-	I have structured the terraform IAC script by dividing it into three main sections for reuse and ease of understanding. Also, I am keeping the script very simple and will not make use of certain functions within it. That is also the reason I am not using Variables.
-	First, the azureauth.tf  --- this tf file will contain the service principal that has been created and granted the required role for us to interact with the ARM provider.
-	Aramark.tf --- this tf file will contain code deploying the Aramark constructs. Ie vng, virtual machines, public ip, resource group, vnic, and virtual network
-	City.tf ---- this tf file will contain code deploying the city constructs. Ie vng, virtual machines, public ip, resource group, vnic, and virtual network
-	The other files are the .tfstate file which is a json file the maintains the state of the resources recorded for deployment by terraform and the terraform.lock.hcl file which records the specific versions of these dependencies that were used during the last successful terraform init execution.

 

Showing resources belonging to our Aramark tenant. Notice the already created connection (IPSec tunnel) that we will use to connect to city.
 
These other resources below belong to the city tenant hence I have put them in a resource group belonging to city.


 

Once the resources have been deployed by our terraform script, we can then go ahead and create our bidirectional connection between our vnets.

 
Creating a vpn connection two sites usually involves two main phases. The first phase is the Handshake or key exchange phase. We use ISAKMP which is a security association protocol that provides mechanisms for devices to authenticate each other before establishing the secure tunnel. This helps ensure you're connecting to a legitimate peer and not a malicious imposter.
          Phase two as I like to describe it, is when we initiate encryption for the tunnel. For encryption. IPSec policies, SHA256 and AES are good options
If you have static routes or are using BGP to advertise routes learnt behind the vpn gateway, you could you access control lists to filter these routes hence providing more secure connections.

         Our terraform script deployed two virtual network gateways, which represent the two endpoints for our IPsec tunnel. As seen below, when we look at our Aramark virtual network gateway, we see connections between both gateways established bidirectionally.

 


Once our IPsec tunnel has been set up successfully, we can go ahead and test for connectivity. This is what our environment now looks like.

 

Now that our tunnel is up and connected, we should be able to test for connectivity and have a successful traffic flow between both vms. We can console into our windows vms and test for connectivity from the terminal. First, lets get more information on our virtual machines. Ip address and hostnames, gateway, and dns ip that is being used to resolve requests.

  

Console into the other one as well so we can confirm our source and destination ip and hostnames.
Note that you can also get this information from the Azure Portal.

We will now begin our testing. Let’s begin with a simple Test-NetConnection to rdp. This confirms that connectivity to the remote vm in city is allowed on port 3389.
Although unconventional, this also confirms that our tunnel is up.

 


Next, we are going to try running pings to the remote site to test for ICMP by using simple ping tests. As you can see, the ping fails. Which indicates that icmp traffic is probably being blocked by a firewall since we already have confirmation that rdp connectivity to the remote site is allowed.
  Furthermore, the terraform script created a NSG (Network security group) attached to the network interface card of both Aramark and city virtual machines. That already narrows the investigation to two filters. First, we check to see if there is a NSG rule that allows ICMP and secondly, we check to see if the firewall on both virtual machines allows for ICMP.

 


As we can see below, I have created a rule in our Network security group to allow for ICMP traffic. A similar rule was created for the NSG attached to the city vm. After creating the rule, we will attempt to reach the other vm again. This time, I am going to run a continuos ping to the remote site. As you can see, it still fails.

 


 

I then decided to do some deep packet inspection by doing a packet capture between the Aramark vm vnic and the city vm’s vnic. 

The wireshark packet capture
We can see that the traffic hits the city vm’s network interface card but does not return so we have no reply from the remote site. This can only mean a block by the guest OS firewall.
As we see below, the windows firewall is on and is probably blocking our icmp requests.

 


 


We can either disable this firewall completely or add a firewall rule in the windows firewall allowing for ping and icmp traffic. Of course, in production or even dev environments, this is not recommended but since we are testing and our purpose is to demonstrate packet inspection, we can turn the firewall off.
Once it is done, we see our pings starting to go through and we are receiving replies from the remote vm.

 
Compared to our previous capture, we are now receiving replies from the remote machine, confirming that that traffic is now allowed to reach the destination.
Notice the ICMP packets captured and also the deep packet analysis between the source and destination.

 

We have now completed our deployment and tested for connectivity across both sites.


![image](https://github.com/Ndollav112711space/TerraformdemoforAFI/assets/132153832/c82db040-04cb-4a08-84d6-ab2baf9917ca)
