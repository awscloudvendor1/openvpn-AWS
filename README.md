## Securely connecting to your AWS Environment using OpenVPN Access Server.

### Introduction
In today's cloud world being able to connect securely and privately to your AWS instances is a necessity. With that said not everyone is able to setup an [AWS Direct Connect](https://aws.amazon.com/directconnect "AWS Direct Connect") connection, or have a network appliance they can setup for [VPN](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpn-connections.html "AWS VPN Connections") connections into AWS.

In this post I will show you how to setup a Software VPN using [OpenVPN](https://openvpn.net/ "OpenVPN") via their AWS Marketplace Offering, setup the local VPN Client to connect to the OpenVPN server, as well as create an instance in a new private subnet in the default VPC that we will use to test our VPN Connectivity. I will also review the costs for having this solution running monthly in your AWS Account.

Our process today will consist of 4 easy steps
1. Launch an EC2 instance from the OpenVPN Access Server AWS Marketplace offering
2. Create an Elastic IP for the OpenVPN Access Server instance and then SSH into the instance to configure the OpenVPN Access Server
3. Create EC2 instance in private subnet to test VPN Connectivity
4. Install the OpenVPN client and connect to the instance in the private subnet

The following are prerequisites for this process:
  * An AWS Account with a default or custom VPC
  * A private subnet in the above mentioned VPC
  * Knowledge of how to install programs as Administrator on Windows, or allow [un-signed applications](http://www.macworld.com/article/3140183/macs/how-to-install-an-app-in-macos-sierra-thats-not-signed-by-a-developer.html) on a Mac

### Step 1 - Launch an EC2 instance from the OpenVPN Access Server AWS Marketplace offering
First we will need to create an EC2 instance using the OpenVPN Marketplace AMI offering.
  1. Login to your AWS account and navigate to the EC2 Dashboard and click, "Launch Instance"
    ![Alt](/assets/1strategy-openvpn-to-aws--ec2-launch-instance.png "Title")

  2. Select "Community AMIs" on the left and then search for "OpenVPN Access Server 2.1.4" Scroll down until you find the AMI for your current region (see table below). I am deploying to us-west-2 so I am looking for ami-d3e743b3. Once you find the appropriate AMI click, "Select"
  ![Alt](/assets/1strategy-openvpn-to-aws--openvpn-ami.png)
  ![Alt](/assets/1strategy-openvpn-to-aws--openvpn-access-server-current-ami-20170208.png)

  3. We are going to use a t2.Micro for our demo which should already be selected for you so click "Next: Configure Instance Details" on the bottom right.

  4. On the *Step 3: Configure Instance Details* page you will see your instance details. Since this is going into the default VPC and a public subnet most settings can be left alone with the exception of *Enable termination protection* as we do not want our VPN being terminated on accident. After clicking the box go ahead and click "Next: Add Storage"

  5. The only thing we will need to change here is make sure the *Volume Type* is set "General Purpose." After checking that go ahead and click "Next: Add Tags"

  6. It's an AWS Best Practice to tag your instances; enter in any tags you want here. At a minimum you should add a value for the default *Name* tag so you can differentiate between instances in the console. I will be using "1Strategy-OpenVPN-Access-Server". After adding any tags click "Next: Configure Security Group"
  ![Alt](/assets/1strategy-openvpn-to-aws--tags.png)

  7. We will be creating a new Security Group for our VPN Server. The ports we will be settings are TCP: 22, 443, 943, and UDP: 1194. After setting your SG access, click "Review and Launch" and then "Launch."
    * TCP
      * port 22   - SSH port
      * port 443  - HTTPS port used for OpenVPN TCP connection
      * port 943  - OpenVPN web-ui
    * UDP
      * port 1194 - OpenVPN UDP port

    *note: All of the default ports can be changed from the admin tool*

  ![Alt](/assets/1strategy-openvpn-to-aws--sgs.png) *note: The warning seen is very important. For this demo I am leaving port 22 open to the world but in a real use case I would limit this to my current IP ONLY.*

  8. After pressing "Launch" you will be presented with the key-pair screen. I will be creating a new key-pair for this demo, but if you already have one, feel free to re-use it. If you are making a new key-pair, type in the name and then click "Download Key Pair" and then "Launch Instances."
  ![Alt](/assets/1strategy-openvpn-to-aws--keypair.png)

  9. Our OpenVPN Access Server is now being created in our AWS Account. Next steps will be to setup an EIP and then SSH into the server to setup OpenVPN.

### Step 2 - Create an Elastic IP for the OpenVPN Instance and then SSH into the instance to configure the OpenVPN Server
Setting an Elastic IP for your instance ensures the VPN Public IP does not change if you need to stop your instances. If it were to change, you would need to reconfigure your server every time.
  1. Login to your AWS account and navigate to the EC2 Dashboard and click "Elastic IPs" on the left. We will be creating a new Elastic IP by clicking "Allocate new address" at the top. Then "Allocate" on the next screen.
  ![Alt](/assets/1strategy-openvpn-to-aws--new-eip.png)
  ![Alt](/assets/1strategy-openvpn-to-aws--new-eip-allocate.png)

  2. Now we will associate our new EIP with the OpenVPN Instance. Select your EIP from the list and then click "Actions>Associate Address."
  3. On the *Associate Address* screen, select your instances from the list and then click "Associate."
  ![Alt](/assets/1strategy-openvpn-to-aws--new-eip-associate.png)

  4. Now that we have an Elastic IP set for our OpenVPN Access Server it's time to SSH into the server. I am using MacOS so I am able to use SSH natively just like on Linux. If you are using Windows you should convert your PEM to a PPK for Putty or look into OpenSSH.

    `$ SSH openvpnas@elastic-ip-here -i key-pair.pem`

  5. On your first time connecting, you will be prompted and asked if to accept the [OpenVPN EULA](https://openvpn.net/index.php/license.html).
  ![Alt](/assets/1strategy-openvpn-to-aws--openvpn-as-eula.png)

  6. After accepting the EULA, the "OpenVPN Access Server Setup Wizard" launches. If you ever need to run the setup wizard again run `$ sudo ovpn-init --ec2` on the server. Here are the prompts you will see and a brief explanation of each.
    * Will this be the primary Access Server node. - Default: yes
      * This is your initial instance of OpenVPN so this should be your primary.
    * Please specify the network interface and IP address to be used by the Admin Web UI. - Default: 2
      * Select 1 here so that we are listening in on all interfaces, unless you have a specific reason not to.
    * Please specify the port number for the Admin Web UI. - Default: 943
      * Only change this if you want to have your Admin Web UI on a different port. note: If you do change this make sure to update your Security Groups to reflect the port change.
    * Please specify the TCP port number for the OpenVPN Daemon. - Default: 443
      * Same as above
    * Should client traffic be routed by default through the VPN? - Default: no
      * If you do this, all of your traffic will be routed through the VPN instance and to the internet. There are many pros and cons of doing this. I set this to "no" and use a 3rd party VPN for my internet traffic, especially when using public/unknown wifi.
    * Should client DNS traffic be routed by default through the VPN? - Default: no
      * Use this ONLY if you want your clients using an on-site DNS server in your AWS Environment.
    * Use local authentication via internal DB?- Default: yes
      * Unless you plan to use a 3rd party authentication like LDAP keep this set to the default.
    * Should private subnets be accessible to clients by default? - Default: yes
      * This is the setting that will allow us to connect to private subnets within our VPC once we have established connection to the VPN.
    * Do you wish to login to the Admin UI as "openvpn"? - Default: yes
      * This is your master admin user. For this demo I will leave it default but in a production use I would change this to something not default for obvious reasons.
    * Please specify your OpenVPN-AS license key (or leave blank to specify later).
      * For now press *Enter* to leave it blank. If you do end up buying a license for extra users you can enter this in the admin tool.  

      The wizard will now complete and you will be left with the web addresses you will use to connect to the Admin Server to
      ![Alt](/assets/1strategy-openvpn-to-aws--openvpn-wizard-done.png)

  7. While still connected via SSH we should create a new password for admin user, and make sure our EC2 instance has all the latest updates.
    * Set a secure password for admin user.  
      `$ sudo passwd openvpn`
    * Make sure our Linux system is update and secure.  
      `$ sudo apt-get update && sudo apt-get upgrade -y`
    * optional: Default timezone is US (Pacific - Los Angeles) if you need to change run this.  
      `$ sudo dpkg-reconfigure tzdata`  

  8. Last thing we need to do before we can connect to the admin area and to our VPN is disable the Source/Destination check in AWS. Without doing this we would not be able to access our private subnets. You can read more about it here. To change this go to the EC2 console in AWS, select your instance, choose *Actions>Networking>Change Source/Dest. Check* as seen below. Choose "Yes, Disable" on the next screen.

    ![Alt](/assets/1strategy-openvpn-to-aws--ec2-disable-check.png)

### Section 3 - Create EC2 instance in private subnet to test VPN Connectivity
To be able to verify and test our VPN connection into our AWS account we will first setup a simple EC2 instance in the private subnet that I mentioned in the prerequisites.
  1. Login to your AWS account and navigate to the EC2 Dashboard and click "Launch Instance" on the left.
  2. Press "Select" next to the top item *Amazon Linux AMI*
  3. On Step 2 leave on t2.micro and click "Next: Configure Instance Details"
  4. On Step 3 make sure to set your Subnet into your private subnet mentioned in the prerequisites. Then click "Review and Launch" as defaults for everything else are fine for this test. *note: this will create a SG for you open to the world make sure you understand this.*  
  5. Step 7 shows you the review of your new instance. Click "Launch" to select your key-pair and then launch the instance.


### Section 4 - Install the OpenVPN client and connect to the instance in the private subnet
Now that we have our OpenVPN Access Server running and an EC2 instance deployed to a private subnet within our VPC it is time to install the OpenVPN Client and test out connectivity.
  1. In your web browser enter the ElasticIP from your OpenVPN Access Server https://elastic-ip-here:943  

  *Note: On your first attempt to connect you will be warned by your browser that the SSL certificate cannot b validated this is OK for our demo but in a real world you will want to setup a real SSL certificate in your setup.*

  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-connect.png)

  2. On the screen enter "openvpn" for the Username and the password you created for the user in Section 2, Step 7.

  3. After your credentials are accepted you will see the screen below. Go ahead and click "Click here to continue" which will download the OpenVPN client installer to your machine.  
  A great thing about this download is that the client already has your connection strings setup for you.  

  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-download.png)

  4. Browse to wherever you download the file (~/Downloads/openvpn-connect-2.1.3.110.dmg for me) and double click on it to start the installation. Follow the on-screen instructions to complete the installation.
  *Note: As mentioned in prerequisites you will need knowledge around installing as an Administrator on Windows or un-signed programs on a Mac.*  

  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-install.png)

  5. After your installation has completed you will find a new icon on your Menu Bar up top. If you click on the icon you will see the Elastic IP of your OpenVPN Access Server instance and an option to connect. Click on "Connect..."  
  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-ready.png)

  6. Enter "openvpn" as the Username, and enter the same password as before and click "Connect".  
  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-login.png)

  You will probably get a notice like the below. This is because the client install came with a configuration file from the OpenVPN Access Server. For this we will go ahead and click "Yes"  

  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-login-profile-warning.png)

  After you are connected you will see the menu bar icon show a green checkmark and a timer next to it. This timer shows how long you have been connected.
  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-client-menu-bar-connected.png)

  7. Now all we have left to do is SSH into the EC2 instance we launched earlier into our private subnet. This will validate our VPN's connectivity into our AWS VPC.  
  `ssh ec2-user@internal-ip-here -i key-pair.pem`

  If successful you will be logged into the instance that resides in your private subnet of your VPC.
  ![ALT](/assets/1strategy-openvpn-to-aws--openvpn-ec2-private-subnet.png)

  8. An optional test would be to disconnect from your OpenVPN connection and try connecting to the instance again.  Your connection should time out. If not make sure your instance is in fact in a private subnet and unaccessible from internet.


#### Summary

In this post, I covered launching an OpenVPN Access Server EC2 instance using the AWS Marketplace offering by OpenVPN, setting up your local client, and connecting to an EC2 instance in a private subnet to verify the VPN is working.

I hope this post helps you out in some way. I plan to expand on this post in the future by showing how to extend your home or small office router into AWS using OpenVPN.

#### About the Author

Justin is a DevOps Engineer at 1Strategy, an AWS Advanced Consulting Partner specializing in Amazon Web Services (AWS). With 17+ years in Information Technology, Justin has accumulated a diversity of hard and soft skills, often dubbed a "Jack-of-all-Trades." Justin is a passionate advocate of gender equality and digital privacy, and serves in the leadership for [Seattle #CoffeeOps.](https://www.meetup.com/Seattle-CoffeeOps/ "Seattle #CoffeeOps")
