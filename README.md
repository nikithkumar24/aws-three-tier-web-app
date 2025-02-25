**AWS Three Tier Web Architecture**

**Description**:
This project is a hands-on walk through of a three-tier web architecture in AWS. We will be manually creating the necessary network, security, app, and database components and configurations in order to run this architecture in an available and scalable manner.

**Audience**:
It is intended for those who have a technical role. The assumption is that you have at least some foundational aws knowledge around VPC, EC2, RDS, S3, ELB and the AWS Console.

**Pre-requisites:
An AWS account. If you don’t have an AWS account, follow the instructions here and click on “Create an AWS Account” button in the top right corner to create one.
IDE or text editor of your choice.
Architecture Overview

In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.

Setup - Part 0
Download Code from Github
**git clone https://github.com/nikithkumar24/aws-three-tier-web-app.git**

S3 Bucket Creation
Navigate to the S3 service in the AWS console and create a new S3 bucket.

Select the region you want and give unique name to the bucket

IAM EC2 Instance Role Creation
Navigate to the IAM dashboard in the AWS console and create an EC2 role.

Select EC2 as the trusted entity.

When adding permissions, include the following AWS managed policies. You can search for them and select them. These policies will allow our instances to download our code from S3 and use Systems Manager Session Manager to securely connect to our instances without SSH keys through the AWS console.

AmazonSSMManagedInstanceCore
AmazonS3ReadOnlyAccess
Give your role a name, and then click Create Role.

Networking and Security - Part 1
Now we will be building out the VPC networking components as well as security groups that will add a layer of protection around our EC2 instances, Aurora databases, and Elastic Load Balancers.

Create an isolated network with the following components:
VPC
Subnets
Route Tables
Internet Gateway
NAT gateway
Security Groups
VPC and Subnets
VPC Creation
Navigate to the VPC dashboard in the AWS console and navigate to Your VPCs on the left hand side.

Make sure VPC only is selected, and fill out the VPC Settings with a Name tag and a CIDR range of your choice.

NOTE: Make sure you pay attention to the region you’re deploying all your resources in. You’ll want to stay consistent for this workshop.

NOTE: Choose a CIDR range that will allow you to create at least 6 subnets.

VPC

Subnet Creation
Next, create your subnets by navigating to Subnets on the left side of the dashboard and clicking Create subnet.

We will need six subnets across two availability zones. That means that three subnets will be in one availability zone, and three subnets will be in another zone. Each subnet in one availability zone will correspond to one layer of our three tier architecture. Create each of the 6 subnets by specifying the VPC we created in part 1 and then choose a name, availability zone, and appropriate CIDR range for each of the subnets.

NOTE: It may be helpful to have a naming convention that will help you remember what each subnet is for. For example in one AZ you might have the following: Public-Web-Subnet-AZ-1, Private-App-Subnet-AZ-1, Private-DB-Subnet-AZ-1.

NOTE: Remember, your CIDR range for the subnets will be subsets of your VPC CIDR range.

Your final subnet setup should be similar to this. Verify that you have 3 subnets across 2 different availability zones.

subnet

Internet Connectivity
Internet Gateway
In order to give the public subnets in our VPC internet access we will have to create and attach an Internet Gateway. Click on Internet Gateway on the Dashboard.

Create your internet gateway by simply giving it a name and clicking Create internet gateway.

After creating the internet gateway, attach it to your VPC. You have a couple options on how to do this, either with the creation success message or the Actions drop down.

internetGateway

Then, select the correct VPC and click Attach internet gateway.

NAT Gateway
In order for our instances in the app layer private subnet to be able to access the internet they will need to go through a NAT Gateway. For high availability, you’ll deploy one NAT gateway in each of your public subnets. Navigate to NAT Gateways on the left side of the current dashboard and click Create NAT Gateway.

Fill in the Name, choose one of the public subnets you have created, and then allocate an Elastic IP. Click Create NAT gateway.

NAT

Repeat step 1 and 2 for the other subnet.

Routing Configuration
Navigate to Route Tables on the left side of the VPC dashboard and click Create route table First, let’s create one route table for the web layer public subnets and name it accordingly.



After creating the route table, you'll automatically be taken to the details page. Scroll down and click on the Routes tab and Edit routes.



Add a route that directs traffic from the VPC to the internet gateway. In other words, for all traffic destined for IPs outside the VPC CDIR range, add an entry that directs it to the internet gateway as a target. Save the changes.



Edit the Explicit Subnet Associations of the route table by navigating to the route table details again. Select Subnet Associations and click Edit subnet associations.



Select the two web layer public subnets you created eariler and click Save associations.



Now create 2 more route tables, one for each app layer private subnet in each availability zone. These route tables will route app layer traffic destined for outside the VPC to the NAT gateway in the respective availability zone, so add the appropriate routes for that.



Once the route tables are created and routes added, add the appropriate subnet associations for each of the app layer private subnets.



Security Groups
Security groups will tighten the rules around which traffic will be allowed to our Elastic Load Balancers and EC2 instances. Navigate to Security Groups on the left side of the VPC dashboard, under Security.

The first security group you’ll create is for the public, internet facing load balancer. After typing a name and description, add an inbound rule to allow HTTP type traffic for your IP.



The second security group you’ll create is for the public instances in the web tier. After typing a name and description, add an inbound rule that allows HTTP type traffic from your internet facing load balancer security group you created in the previous step. This will allow traffic from your public facing load balancer to hit your instances. Then, add an additional rule that will allow HTTP type traffic for your IP. This will allow you to access your instance when we test.



The third security group will be for our internal load balancer. Create this new security group and add an inbound rule that allows HTTP type traffic from your public instance security group. This will allow traffic from your web tier instances to hit your internal load balancer. 

The fourth security group we’ll configure is for our private instances. After typing a name and description, add an inbound rule that will allow TCP type traffic on port 4000 from the internal load balancer security group you created in the previous step. This is the port our app tier application is running on and allows our internal load balancer to forward traffic on this port to our private instances. You should also add another route for port 4000 that allows your IP for testing. 

The fifth security group we’ll configure protects our private database instances. For this security group, add an inbound rule that will allow traffic from the private instance security group to the MYSQL/Aurora port (3306). 

Database Deployment - Part 2
Deploy Database Layer
Subnet Groups
Multi-AZ Database
Subnet Groups
Navigate to the RDS dashboard in the AWS console and click on Subnet groups on the left hand side. Click Create DB subnet group.
Give your subnet group a name, description, and choose the VPC we created. 
When adding subnets, make sure to add the subnets we created in each availability zone specificaly for our database layer. You may have to navigate back to the VPC dashboard and check to make sure you're selecting the correct subnet IDs. 
Multi-AZ Database Deployment
Navigate to Databases on the left hand side of the RDS dashboard and click Create database.

We'll now go through several configuration steps. Start with a Standard create for this MySQL-Compatible Amazon Aurora database. Leave the rest of the defaults in the Engine options as default.



Under the Templates section choose Dev/Test since this isn't being used for production at the moment. Under Settings set a username and password of your choice and note them down since we'll be using password authentication to access our database.



Next, under Availability and durability change the option to create an Aurora Replica or reader node in a different availability zone. Under Connectivity, set the VPC, choose the subnet group we created earlier, and select no for public access.



Set the security group we created for the database layer, make sure password authentication is selected as our authentication choice, and create the database.



When your database is provisioned, you should see a reader and writer instance in the database subnets of each availability zone. Note down the writer endpoint for your database for later use. 

App Tier Instance Deployment - Part 3
In this section, we will create an EC2 instance for our app layer and make all necessary software configurations so that the app can run. The app layer consists of a Node.js application that will run on port 4000. We will also configure our database with some data and tables.

Learning Objectives:
Create App Tier Instance
Configure Software Stack
Configure Database Schema
Test DB connectivity
App Instance Deployment
Navigate to the EC2 service dashboard and click on Instances on the left hand side. Then, click Launch Instances.

Select the first Amazon Linux 2 AMI

We'll be using the free tier eligible T.2 micro instance type. Select that and click Next: Configure Instance Details.

When configuring the instance details, make sure to select to correct Network, subnet, and IAM role we created. Note that this is the app layer, so use one of the private subnets we created for this layer.



We'll be keeping the defaults for storage so click next twice. When you get to the tag screen input a Name as a key and call the instance AppLayer. It's a good idea to tag your instances so you can easily keep track of what each instance was created for. Click Next: Configure Security Group.



Earlier we created a security group for our private app layer instances, so go ahead and select that in this next section. Then click Review and Launch. Ignore the warning about connecting to port 22- we don't need to do that.

When you get to the Review Instance Launch page, review the details you configured and click Launch. You'll see a pop up about creating a key pair. Since we are using Systems Manager Session Manager to connect to the instance, proceed without a keypair. Click Launch. You'll be taken to a page where you can click launch instance, and you'll see the instance you just launched.

Connect to Instance
Navigate to your list of running EC2 Instances by clicking on Instances on the left hand side of the EC2 dashboard. When the instance state is running, connect to your instance by clicking the checkmark box to the left of the instance, and click the connect button on the top right corner of the dashboard.Select the Session Manager tab, and click connect. This will open a new browser tab for you.

NOTE: If you get a message saying that you cannot connect via session manager, then check that your instances can route to your NAT gateways and verify that you gave the necessary permissions on the IAM role for the Ec2 instance.

When you first connect to your instance like this, you will be logged in as ssm-user which is the default user. Switch to ec2-user by executing the following command in the browser terminal:

sudo -su ec2-user
Let’s take this moment to make sure that we are able to reach the internet via our NAT gateways. If your network is configured correctly up till this point, you should be able to ping the google DNS servers:

ping 8.8.8.8
You should see a transmission of packets. Stop it by pressing Cntrl+C

NOTE: If you can’t reach the internet then you need to double check your route tables and subnet associations to verify if traffic is being routed to your NAT gateway!

Configure Database
Start by downloading the MySQL CLI:

sudo yum install mysql -y
Initiate your DB connection with your Aurora RDS writer endpoint. In the following command, replace the RDS writer endpoint and the username, and then execute it in the browser terminal:

mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
You will then be prompted to type in your password. Once you input the password and hit enter, you should now be connected to your database.

NOTE: If you cannot reach your database, check your credentials and security groups.

Create a database called webappdb with the following command using the MySQL CLI:

CREATE DATABASE webappdb;
You can verify that it was created correctly with the following command:

SHOW DATABASES;
Create a data table by first navigating to the database we just created:

USE webappdb;
Then, create the following transactions table by executing this create table command:

CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));
Verify the table was created:

SHOW TABLES;
Insert data into table for use/testing later:

INSERT INTO transactions (amount,description) VALUES ('400','groceries');
Verify that your data was added by executing the following command:

SELECT * FROM trasactions;
When finished, just type exit and hit enter to exit the MySQL client.

Configure App Instance
The first thing we will do is update our database credentials for the app tier. To do this, open the application-code/app-tier/DbConfig.js file from the github repo in your favorite text editor on your computer. You’ll see empty strings for the hostname, user, password and database. Fill this in with the credentials you configured for your database, the writer endpoint of your database as the hostname, and webappdb for the database. Save the file.

NOTE: This is NOT considered a best practice, and is done for the simplicity of the lab. Moving these credentials to a more suitable place like Secrets Manager is left as an extension for this project.

Upload the app-tier folder to the S3 bucket that you created in part 0.

Go back to your SSM session. Now we need to install all of the necessary components to run our backend application. Start by installing NVM (node version manager).

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
Next, install a compatible version of Node.js and make sure it's being used

nvm install 16
nvm use 16
PM2 is a daemon process manager that will keep our node.js app running when we exit the instance or if it is rebooted. Install that as well.

npm install -g pm2
Now we need to download our code from our s3 buckets onto our instance. In the command below, replace BUCKET_NAME with the name of the bucket you uploaded the app-tier folder to:

cd ~/
aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
Navigate to the app directory, install dependencies, and start the app with pm2.

cd ~/app-tier
npm install
pm2 start index.js
To make sure the app is running correctly run the following:

pm2 list
If you see a status of online, the app is running. If you see errored, then you need to do some troubleshooting. To look at the latest errors, use this command:

pm2 logs
NOTE: If you’re having issues, check your configuration file for any typos, and double check that you have followed all installation commands till now.

Right now, pm2 is just making sure our app stays running when we leave the SSM session. However, if the server is interrupted for some reason, we still want the app to start and keep running. This is also important for the AMI we will create:

pm2 startup
After running this you will see a message similar to this.

[PM2] To setup the Startup Script, copy/paste the following command: sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.0.0/bin /home/ec2-user/.nvm/versions/node/v16.0.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user —hp /home/ec2-user
DO NOT run the above command, rather you should copy and past the command in the output you see in your own terminal. After you run it, save the current list of node processes with the following command:

pm2 save
Test App Tier
Now let's run a couple tests to see if our app is configured correctly and can retrieve data from the database.

To hit out health check endpoint, copy this command into your SSM terminal. This is our simple health check endpoint that tells us if the app is simply running.

curl http://localhost:4000/health
The response should looks like the following:

"This is the health check"
Next, test your database connection. You can do that by hitting the following endpoint locally:

curl http://localhost:4000/transaction
You should see a response containing the test data we added earlier:

{
  "result": [
    { "id": 1, "amount": 400, "description": "groceries" },
    { "id": 2, "amount": 100, "description": "class" },
    { "id": 3, "amount": 200, "description": "other groceries" },
    { "id": 4, "amount": 10, "description": "brownies" }
  ]
}
If you see both of these responses, then your networking, security, database and app configurations are correct.

Congrats! Your app layer is fully configured and ready to go.

Internal Load Balancing and Auto Scaling
In this section,we will create an Amazon Machine Image (AMI) of the app tier instance we just created, and use that to set up autoscaling with a load balancer in order to make this tier highly available.

Learning Objectives:
Create an AMI of our App Tier
Create a Launch Template
Configure Autoscaling
Deploy Internal Load Balancer
App Tier AMI
Navigate to Instances on the left hand side of the EC2 dashboard. Select the app tier instance we created and under Actions select Image and templates. Click Create Image.



Give the image a name and description and then click Create image. This will take a few minutes, but if you want to monitor the status of image creation you can see it by clicking AMIs under Images on the left hand navigation panel of the EC2 dashboard.



Target Group
While the AMI is being created, we can go ahead and create our target group to use with the load balancer. On the EC2 dashboard navigate to Target Groups under Load Balancing on the left hand side. Click on Create Target Group.

The purpose of forming this target group is to use with our load blancer so it may balance traffic across our private app tier instances. Select Instances as the target type and give it a name.

Then, set the protocol to HTTP and the port to 4000. Remember that this is the port our Node.ja app is running on. Select the VPC we've been using thus far, and then change the health check path to be /health. This is the health check endpoint of our app. Click Next.

We are NOT going to register any targets for now, so just skip that step and create the target group.

Internal Load Balanceer
On the left hand side of the EC2 dashboard select Load Balancers under Load Balancing and click Create Load Balancer.

We'll be using an Application Load Balancer for our HTTP traffic so click the create button for that option.

After giving the load balancer a name, be sure to select internal since this one will not be public facing, but rather it will route traffic from our web tier to the app tier.



Select the correct network configuration for VPC and private subnets.



Select the security group we created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our target group that we just created, so select it from the dropdown, and create the load balancer.



Launch Template
Before we configure Auto Scaling, we need to create a Launch template with the AMI we created earlier. On the left side of the EC2 dashboard navigate to Launch Template under Instances and click Create Launch Template.

Name the Launch Template, and then under Application and OS Images include the app tier AMI you created.



Under Instance Type select t2.micro. For Key pair and Network Settings don't include it in the template. We don't need a key pair to access our instances and we'll be setting the network information in the autoscaling group.



Set the correct security group for our app tier, and then under Advanced details use the same IAM instance profile we have been using for our EC2 instances.





Auto Scaling
We will now create the Auto Scaling Group for our app instances. On the left side of the EC2 dashboard navigate to Auto Scaling Groups under Auto Scaling and click Create Auto Scaling group.

Give your Auto Scaling group a name, and then select the Launch Template we just created and click next. 

On the Choose instance launch options page set your VPC, and the private instance subnets for the app tier and continue to step 3.



For this next step, attach this Auto Scaling Group to the Load Balancer we just created by selecting the existing load balancer's target group from the dropdown. Then, click next.



For Configure group size and scaling policies, set desired, minimum and maximum capacity to 2. Click skip to review and then Create Auto Scaling Group.

You should now have your internal load balancer and autoscaling group configured correctly. You should see the autoscaling group spinning up 2 new app tier instances. If you wanted to test if this is working correctly, you can delete one of your new instances manually and wait to see if a new instance is booted up to replace it.

NOTE: Your original app tier instance is excluded from the ASG so you will see 3 instances in the EC2 dashboard. You can delete your original instance that you used to generate the app tier AMI but it's recommended to keep it around for troubleshooting purposes.

Web Tier Instance Deployment - Part 5
In this section we will deploy an EC2 instance for the web tier and make all necessary software configurations for the NGINX web server and React.js website.

Learning Objectives
Update NGINX Configuration Files
Create Web Tier Instance
Configure Software Stack
Update Config File
Before we create and configure the web instances, open up the application-code/nginx.conf file from the repo we downloaded. Scroll down to line 58 and replace [INTERNAL-LOADBALANCER-DNS] with your internal load balancer’s DNS entry. You can find this by navigating to your internal load balancer's details page.



Then, upload this file and the application-code/web-tier folder to the s3 bucket you created for this project.

Web Instance Deployment
Follow the same instance creation instructions we used for the App Tier instance in Part 3: App Tier Instance Deployment, with the exception of the subnet. We will be provisioning this instance in one of our public subnets. Make sure to select the correct network components, security group, and IAM role. This time, auto-assign a public ip on the Configure Instance Details page. Remember to tag the instance with a name so we can identify it more easily.





Then at the end, proceed without a key pair for this instance.

Connect to Instance
Follow the same steps you used to connect to the app instance and change the user to ec2-user. Test connectivity here via ping as well since this instance should have internet connectivity:

sudo -su ec2-user
ping 8.8.8.8
Note: If you don't see a transfer of packets then you'll need to verify your route tables attached to the subnet that your instance is deployed in.

Configure Web Instance
We now need to install all of the necessary components needed to run our front-end application. Again, start by installing NVM and node

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
Now we need to download our web tier code from our s3 bucket:

cd ~/
aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
Navigate to the web-layer folder and create the build folder for the react app so we can serve our code:

cd ~/web-tier
npm install
npm run build
NGINX can be used for different use cases like load balancing, content caching etc, but we will be using it as a web server that we will configure to serve our application on port 80, as well as help direct our API calls to the internal load balancer.

sudo amazon-linux-extras install nginx1 -y
We will now have to configure NGINX. Navigate to the Nginx configuration file with the following commands and list the files in the directory:

cd /etc/nginx
ls
You should see an nginx.conf file. We’re going to delete this file and use the one we uploaded to s3. Replace the bucket name in the command below with the one you created for this project:

sudo rm nginx.conf
sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .
Then, restart Nginx with the following command:

sudo service nginx restart
To make sure Nginx has permission to access our files execute this command:

chmod -R 755 /home/ec2-user
And then to make sure the service starts on boot, run this command:

sudo chkconfig nginx on
Now when you plug in the public IP of your web tier instance, you should see your website, which you can find on the Instance details page on the EC2 dashboard. If you have the database connected and working correctly, then you will also see the database working. You’ll be able to add data. Careful with the delete button, that will clear all the entries in your database.





External Load Balancer and Auto Scaling - Part 6
In this section of the workshop we will create an Amazon Machine Image (AMI) of the web tier instance we just created, and use that to set up autoscaling with an external facing load balancer in order to make this tier highly available.

Learning Objectives:
Create an AMI of our Web Tier
Create a Launch Template
Configure Auto Scaling
Deploy External Load Balancer
Web Tier AMI
Navigate to Instances on the left hand side of the EC2 dashboard. Select the web tier instance we created and under Actions select Image and templates. Click on Create Image.

Give the image a name and description and then click Create image. This will take a few minutes, but if you want to monitor the status of image creation you can see it by clicking AMIs under Images on the left hand navigation panel of the EC2 dashboard.

Target Group
While the AMI is being created, we can go ahead and create our target group to use with the load balancer. On the EC2 dashboard navigate to Target Groups under Load Balancing on the left hand side. Click on Create Target Group.

The purpose of forming this target group is to use with our load blancer so it may balance traffic across our public web tier instances. Select Instances as the target type and give it a name.

Then, set the protocol to HTTP and the port to 80. Remember this is the port NGINX is listening on. Select the VPC we've been using thus far, and then change the health check path to be /health. Click Next.

We are NOT going to register any targets for now, so just skip that step and create the target group.

Internet Facing Load Balancer
On the left hand side of the EC2 dashboard select Load Balancers under Load Balancing and click Create Load Balancer.

We'll be using an Application Load Balancer for our HTTP traffic so click the create button for that option.

After giving the load balancer a name, be sure to select internet facing since this one will not be public facing, but rather it will route traffic from our web tier to the app tier.

Select the correct network configuration for VPC and public subnets.

Select the security group we created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our target group that we just created, so select it from the dropdown, and create the load balancer.

Launch Template
Before we configure Auto Scaling, we need to create a Launch template with the AMI we created earlier. On the left side of the EC2 dashboard navigate to Launch Template under Instances and click Create Launch Template.

Name the Launch Template, and then under Application and OS Images include the app tier AMI you created.



Under Instance Type select t2.micro. For Key pair and Network Settings don't include it in the template. We don't need a key pair to access our instances and we'll be setting the network information in the autoscaling group.

Set the correct security group for our web tier, and then under Advanced details use the same IAM instance profile we have been using for our EC2 instances.

Auto Scaling
We will now create the Auto Scaling Group for our web instances. On the left side of the EC2 dashboard navigate to Auto Scaling Groups under Auto Scaling and click Create Auto Scaling group.

Give your Auto Scaling group a name, and then select the Launch Template we just created and click next.

On the Choose instance launch options page set your VPC, and the public subnets for the web tier and continue to step 3.

For this next step, attach this Auto Scaling Group to the Load Balancer we just created by selecting the existing web tier load balancer's target group from the dropdown. Then, click next.

For Configure group size and scaling policies, set desired, minimum and maximum capacity to 2. Click skip to review and then Create Auto Scaling Group.

You should now have your external load balancer and autoscaling group configured correctly. You should see the autoscaling group spinning up 2 new web tier instances. If you wanted to test if this is working correctly, you can delete one of your new instances manually and wait to see if a new instance is booted up to replace it. To test if your entire architecture is working, navigate to your external facing loadbalancer, and plug in the DNS name into your browser

NOTE: Again, your original web tier instance is excluded from the ASG so you will see 3 instances in the EC2 dashboard. You can delete your original instance that you used to generate the web tier AMI but it's recommended to keep it around for troubleshooting purposes.



** Congrats! You’ve Implemented a 3 Tier Web Architecture! **
Clean Up
In order to clean up the resources in the AWS account and avoid incurring any other charges, you'll want to delete the resources in a specific order. We will start with the different services deployed inside the VPC, and then the VPC components themselves:

Autoscaling Groups

We will delete the Auto Scaling Groups first since they will continue to deploy EC2 instances if we delete them. This is because we had set the min number of instances to 2 at every layer. In the EC2 console dashboard you can simply select your app and web tier Auto Scaling Groups and click on the delete button.

Application Load Balancers

For each of the Application Load Balancers we created, start by deleting the listeners. Then, delete the load balancers by checking the box next to them, and then clicking delete under the Actions dropdown.

Target Groups

Now that there's no Application Load Balancer dependency on target groups we can delete them. Navigate to Target Groups on the left hand side of the Ec2 dashboard. Select your target groups and click delete under the Actions dropdown.

Launch Templates

Navigate to Launch Templates on the left hand side of the EC2 dashboard. Select a template and click the Delete template button under the Actions dropdown. You will need to type Delete in the popup that appears.

AMIs

Navigate to AMIs under Images on the left hand side of the EC2 dashboard. Select both of the AMIs and click Deregister AMI under the Actions dropdown.

Remaining EC2 Instances

If you deleted the instances you used to create you AMIs you may skip this step. Otherwise, navigate to Instances under Instances in the left hand side of the EC2 dashboard. Select all instances and click Terminate Instance under the Instance State dropdown.

Aurora Database

To delete the database we created, navigate to the RDS service dashboard. We need to make sure that deletion protection is off. Do this by selecting the regional cluster and clicking the Modify button. Scroll all the way down and you should be able to uncheck deletion protection if it is checked.

On the next screen, select Apply Immidiately and then click Modify Cluster.

Now we can delete the reader and writer nodes separately. To do that, select one of them and click delete from the Actions dropdown. You'll have to type delete me into the resulting pop-up. Repeat for the other database instance. For the last instance the pop up with ask to create a snapshot. Don't do this and check off the acknowledgement.

Nat Gateways

Navigate to the VPC service dashboard. On the left hand side select NAT Gateways and delete both of the nat gateways we created by selecting them and clicking delete NAT gateway. You will need to type the work delete into the pop up to confirm the deletion.

Elastic IPs

Release any Elasic IPs that were allocated by navigating to Elastic IPs under Virtual Private Cloud on the left hand side of the VPC dashboard. Select all of the Elastic IPs and click Release Elastic IP addresses under the Actions dropdown. Click Release on the resulting pop-up.

Route tables

To delete the route table we created, navigate to Route Tables on the left hand side of the VPC dashboard. First, we need to remove all subnet associations for each route table. Select one of your route tables you created and choose Subnet Association in the bottom window pane displaying route table details. Then, under Explicit subnet associations click Edit subnet associations.

Then, uncheck all of the associated subnets and save the associations.

Once you've removed all route table subnet association, select all three of the route tables we created and click delete under the Actions dropdown. There will be another pop up here- just type in the word delete.

Internet Gateway

Navigate to Internet Gateways on the left hand side of the VPC dashboard. Start by selecting the Internet Gateway we created and click Detach from VPC. Then, click Detach Internet Gateway on the pop-up.

After detaching the Internet Gateway, select it and click Delete internet gateway from the actions dropdown. You'll have to type in the work delete in the resulting pop up.

VPC

Lastly, delete the VPC by navigating to Your VPCs on the lefthand side of the VPC dashboard. Select the VPC that we created for this lab, and under the Actions dropdown click Delete VPC. This will also delete the subnets and security groups.

Thank You😊
