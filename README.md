# Project-3-MIGRATION-OF-BITBUCKET-DATA-CENTER-FROM-ONE-REGION-TO-ANOTHER-REGION-LEVERAGING-MGN-IN-AWS

## Migration of Bitbucket Data from one region(Source:N. Virginia) to another(Target:Oregon)
## Prerequisites
AWS Account: Ensure you have access to the AWS account in which the migration will be executed 
Bitbucket Data Center: Ensure you have administrative access to your existing Bitbucket Data Center.
**AWS Application Migration Service (MGN)**: Verify that MGN is enabled in both the source and target regions.
**Networking**: Ensure proper VPC, subnets, security groups, and routing are configured in both regions to support Bitbucket’s requirements.
**Backups**: Create a full backup of your Bitbucket Data Center (database, shared storage, and configuration files) before migration.

## Architecture

- Source Region (us-east-1):- EC2 instances hosting the Bitbucket application and data.

- Target Region (us-west-2):- Replicated EC2 instances for hosting Bitbucket data after migration.

- AWS MGN:- Automates replication and server launch.

- DNS Update:- Redirect traffic to servers in Oregon.
  
  
<img width="794" alt="Image" src="https://github.com/user-attachments/assets/51628417-02ca-48fd-a407-e8e2594708be" />


## Bitbucket Data Center in AWS:

- Log in to the AWS Management Console and launch an instance. use instance type:`t2.medium`, amazon Machine image:`linux 2AMI`, Configure storage:`50GB`, Select a `keypair`, select a `VPC` and `Subnet`, `Enable` auto-assign public IP, Allow traffic from: port `22`(ssh), `7990`(bitbucket) .

<img width="899" alt="Image" src="https://github.com/user-attachments/assets/8a180850-558d-4f95-89d2-af899ab33f3a" />

## Add a user data on Advance details
-The user data will be utilized to update the system, install Docker, set up a directory for storing Bitbucket data, and deploy the Bitbucket Docker container on the server.
```
#!/bin/bash
# Update the system
sudo yum update -y

# Install Docker
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user

# Create a directory for Bitbucket data
sudo mkdir -p /var/atlassian/application-data/bitbucket
sudo chown -R ec2-user:ec2-user /var/atlassian/application-data/bitbucket

# Run Bitbucket Docker container
docker run -v /var/atlassian/application-data/bitbucket:/var/atlassian/application-data/bitbucket \
  --name="bitbucket" \
  -d \
  -p 7990:7990 \
  -p 7999:7999 \
  atlassian/bitbucket-server

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker
```
- select `Launch instance` to launch the server.
- select the instance and use `EC2 instance connect` or `ssh client(Git bash)` to connect to the server on the terminal. 
  
- On your terminal Run
- SSH into your Linux EC2 instance in your source region and check if docker is install and active using this commands:
```
docker ps -a #to see the containers.
```

<img width="467" alt="Image" src="https://github.com/user-attachments/assets/b85daaf8-27fb-4164-8462-8f9677573a70" />


- Check if docker is active by running 
```
sudo systemctl status docker
```


<img width="482" alt="Image" src="https://github.com/user-attachments/assets/261fc806-326e-4fea-8dd5-b358e9c5bd08" />


- On your server, select the EC2 instance and copy the public ip address and paste on your brower adding the port number e.g `52.7.182.131:7990` to access it.

 
![Image](https://github.com/user-attachments/assets/936a8451-0abb-4fe3-b920-2498c26fec82)

# Perform the Migration Using MGN

# Intallation of replication agent
- Use aws-mgn application migration service in the target region (Oregon)
- Navigate to the AWS Application Migration Service (MGN) section.
  
- **Initialize the AWS MGN**

-In order to use Application Migration Service for the first time, the service must first be initialize for any AWS region in which you plan to use MGN and copy the server over i.e. your Target region(us-west-2).
- Go to `AWS MGN` => `Getting Started` => `Set up an Application Migration Service`
- leave all the option as it is and click `Add Server`
  

**AWS Replication Agent installation**
  on MGN
- Select you operating system `Linux` or `windows`
- Select your replication preference `all disks` (Enter to replicated all disks)
- Add your `access key ID` and `secret Access key` gotten from `IAM User`. if you don't have you can generate it as follows
  
- **Generate the required AWS credentials**
- Go to `IAM` => `User` => `User` `#if you don't have a user create it and add, provide 'user name' and select 'Access type' as Programmatic'`
- Choose the `Attach existing policies` directly and attach – `AWSApplicationMigrationAgentPolicy`

    <img width="931" alt="Image" src="https://github.com/user-attachments/assets/23caf94a-3e89-4e58-9a1d-a4543e9afb15" />
    
- Click on `User` in the left-hand pane, then click the user's name. Copy the Access Key ID and Secret Access Key.
  - Past the IAM keys on MGN were required.
  
<img width="904" alt="Image" src="https://github.com/user-attachments/assets/035d14e8-e41b-43fb-b19f-d67f05cfe407" />


  - Download the installer by copying this command and pasting on you source server on the terminal, make sure you Replace <AWS_REGION> with the region where AWS MGN is configured (e.g., us-west-2 target region)
 
   ```
  sudo wget -O ./aws-replication-installer-init https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
   ```

 - Make the installer executable and Run the installer with root privileges by Copy and input the command below into the command line on your source server. 

  ```
  sudo chmod +x aws-replication-installer-init; sudo ./aws-replication-installer-init --region us-west-2 # Target region Oregon
  ```

  <img width="713" alt="Image" src="https://github.com/user-attachments/assets/fb40e040-9c63-4fc4-98ec-f0ed16c32866" />
  

 **Verify Installation**
- Check the AWS MGN Console in the target Servers section:
The target server should appear as "Pending" or "Ready for replication."
- Select `Back`
- Select `source server name`
  
- Monitor the replication as it moves from `Not ready to ready to test, test in progress, ready for cutover, cutover in progress, cutover complete`
- On the migration dash board, the lifecycle indicate `not ready`
- Once data replication reaches 100%, the server status will change to `Ready for Test`.
  
  
![Image](https://github.com/user-attachments/assets/317e5a40-78d2-4941-bcbc-9c0bcd84aeb3)


  **Launch the Test Instance**

  - Select the `Source Servers` => `Test and Cutover` => `Launch Test Instance`

  - A test instance should be launched as per your Launch Template
  - Click on the `Source Servers` => Lifecycle => Click on the `job id` and monitor the progress

   - Go to EC2 => A new AWS Application Migration Service Conversion server should be launched

  - Once the conversion is complete a test instance should be available

  - Test the application by browsing using the test instances’ public IP
  - If the Public IP is not availble then;
  - Creat an Elastic IP address
  - Go to the EC2 internal instance and select it
  - Select `elastic Ip` at the left
  - Click `Allocate elastic ip address`
  - Select the allocated Ip and take `associate IP`
  - Chose the `instance` and the `private ip`
  - Also attached a `security group` to access it openning the right ports e.g `7990`
    
    <img width="907" alt="Image" src="https://github.com/user-attachments/assets/0744e486-ab8e-4e40-ac13-f0346e240a20" />
    <img width="955" alt="Image" src="https://github.com/user-attachments/assets/9307de85-9a6a-4121-8952-e7d6bbe5ba0c" />

    - An IP internal main testing server is created, you can test this instance in the internet.
      
![Image](https://github.com/user-attachments/assets/936a8451-0abb-4fe3-b920-2498c26fec82)
    
 
 - On MGN source server
  - click  `test and cutover` and select `mark as ready for cutover` then take continue. This is do the final migration, will create a new server for your actual migration. the cutover will delete the replication server.

    <img width="884" alt="Image" src="https://github.com/user-attachments/assets/ad0634e4-179d-4020-85c2-db3a77e056e2" />

    - The replication instances are terminated

     
      <img width="928" alt="Image" src="https://github.com/user-attachments/assets/ad2cf711-ffaa-4429-ad1c-3277da70279e" />

 
  - Perform a Test Cutover
  - Select `test and cutover` and select `launch cutover instance` and select `launch`; this will create another server to replicate the serve.

 <img width="884" alt="Image" src="https://github.com/user-attachments/assets/82585c51-1bf2-478e-a065-d9d997c3da15" />
 
  -   Cutover in progress 
  - On the Source Servers, select the server `marked Ready for Cutover`.


    **launch the cutover instance** 

- Select the `source Servers` => `Test and cutover` => `Launch Cutover instance`
- A cutover instance should be launched as per your launch template
- Go to EC2 => A new AWS MGN service conversion server should be launch 
- Once this is completed a cutover instance should be available
- Review and confirm the instance settings, then click `Launch`.

**Expected** Result: The cutover instance is launched in AWS, and the server enters the Cutover in Progress state. This step involves transitioning production traffic to the new environment.

<img width="887" alt="Image" src="https://github.com/user-attachments/assets/f30b539f-95c6-4573-b569-23a7c4dabcb9" />

 - New servers (IP internal, webapp-server, AWS application Migration Service and Replication server) are created with with all the data inside
 - 
   <img width="928" alt="Image" src="https://github.com/user-attachments/assets/f959b95a-c1ad-4513-9221-6a56238ed7f8" />
   
   - Test the newly Internal server by associating the elastic ip and security group before testing on the server.
   
- Cutover Complete
  - After verifying that the cutover instance is operating correctly, return to the Source Servers page.
  - Click `test and cutover` `finalize cutover` # all the servers involved in the migration and replication data will be deleted acept you last application server. 
  - click `Mark as Cutover Complete`.

**Expected Result:** The server status changes to Cutover Complete, signifying that the migration is finished. The cutover instance is now fully operational, and the source server can be archived (take `action` and select `mark as archived` ) or decommissioned as needed.
- Migration successful to the new region Oregon us-west-2

<img width="908" alt="Image" src="https://github.com/user-attachments/assets/f2b11b86-192c-44ea-bb4d-fd02a3045fc8" />


## Testing your replicated server in the Target region.
- Open AWS MGN console in the target region (Oregon)
- check replication status is healthy and launch the test instance
- Access the test instance in EC2 and connect to it via SSH and port 7990 to get to the bitbucket server.
- verify that the application is working and the data looks consistent
  
## Conclusion
This project successfully outlines the migration of Bitbucket data from the AWS N Virginia region to the Oregon region using AWS Application Migration Service (MGN). By following the AWS Console procedures, the migration ensures minimal downtime and preserves data integrity. This process is ideal for organizations looking to transition workloads across AWS regions.


## contribution
- More ideas on region migration are welcome!

  HAPPY LEARNING 
