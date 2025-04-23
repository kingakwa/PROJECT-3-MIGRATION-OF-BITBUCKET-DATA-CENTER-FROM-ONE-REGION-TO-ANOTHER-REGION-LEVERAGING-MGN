# Project-3-MIGRATION-OF-BITBUCKET-DATA-CENTER-FROM-ONE-REGION-TO-ANOTHER-REGION-LEVERAGING-MGN-IN-AWS

## MIGRATION-OF-BITBUCKET-DATA-from-one-region(N Virginia)-to-another(Oregon)
## Prerequisites
AWS Account: Ensure you have access to the AWS accounts for both source and target regions.
Bitbucket Data Center: Ensure you have administrative access to your existing Bitbucket Data Center.
AWS Application Migration Service (MGN): Verify that MGN is enabled in both the source and target regions.
Networking: Ensure proper VPC, subnets, security groups, and routing are configured in both regions to support Bitbucket’s requirements.
Backups: Create a full backup of your Bitbucket Data Center (database, shared storage, and configuration files) before migration.

 Bitbucket Data Center in AWS:
Log in to the AWS Management Console. use t2medium, linux 2, 50GB, keypair, port 22(ssh), 7990(bitbucket) 


#user data 
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
```


run 
docker ps to see the containers. 

run this to check if docker is actice and running 

```
sudo systemctl status docker
```
<img width="626" alt="Image" src="https://github.com/user-attachments/assets/0a36a9ce-8756-43cc-803b-10f85ff90da5" />

copy the public ip address of the instance and add the port number e.g `52.7.182.131:7990`
<img width="650" alt="Image" src="https://github.com/user-attachments/assets/9e205dc6-4f26-459d-bab5-5203c179eeb4" />

# Perform the Migration Using MGN
## Start Replication:
In the MGN console, start the replication process.
AWS MGN will create staging resources in the target region to replicate the source instances.

# intallation of application on agent
- use aws-mgn application migration service in the target region (Oregon)
- Navigate to the AWS Application Migration Service (MGN) section.
-In the Source Servers section, click `Add Source Server`.
- Download the Replication Agent installer for your operating system
- select `Linux` or `windows`
- add `the access key` and `secret key` gotten from AIM user from aws
<img width="904" alt="Image" src="https://github.com/user-attachments/assets/f90a0550-aec8-40fb-916e-aafbb33cd27f" />


- Download the installer using this command on you source terminal
- 
   ```
  sudo wget -O ./aws-replication-installer-init https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
   ```

  Make the installer executable and Run the installer with root privileges using this command, make sure Replace <AWS_REGION> with the region where AWS MGN is configured (e.g., us-east-1 target region)
  
  ```
  sudo chmod +x aws-replication-installer-init; sudo ./aws-replication-installer-init --region us-east-1
  ```

  <img width="713" alt="Image" src="https://github.com/user-attachments/assets/fb40e040-9c63-4fc4-98ec-f0ed16c32866" />
this will install the application agent

- Verify Installation
After installation, the replication agent will register the source server with AWS MGN.
-Check the AWS MGN Console in the Source Servers section:
The source server should appear as "Pending" or "Ready for replication."

-select `Back`
-select `source server`

-monitor the replication as it moves from `Not ready to ready to test, test in progress, ready for cutover, cutover in progress, cutover complete`
<img width="923" alt="Image" src="https://github.com/user-attachments/assets/a6306f0d-9540-4e12-a21f-de0548e8e4bd" />
- Ready for testing.
  click `test and cutover` then select `launch test instance`
   an IP internal main testing server is form, you can test this instance.
  if it does not have a public address, create one using `Elastic IP`

  -test in progress
  *creat an Elastic IP address
  - Go to the ec2 internal instance and select it
  - select `elastic Ip` at the left
  - click `Allocate elastic ip address`
  - select the allocated Ip and take `associate IP`
  - chose the `instance` and the `private ip`
  - also attached a `security group` to access the it openning the right ports
 <img width="907" alt="Image" src="https://github.com/user-attachments/assets/f7439008-fe8c-4619-abe3-3a3aee99af3a" />
  - click  `test and cutover` and select `mark as ready for cutover` then take continue
 
  - Perform a Test Cutover
  - select test and cutover and select launch cutover instance and select launch; this will create another server to replicate the serve.
 
  - cutover in progress 
On the Source Servers page, select the server marked Ready for Cutover.

- Click Launch Cutover Instances.

- Review and confirm the instance settings, then click Launch.

Expected Result: The cutover instance is launched in AWS, and the server enters the Cutover in Progress state. This step involves transitioning production traffic to the new environment.

- Cutover Complete
- fter verifying that the cutover instance is operating correctly, return to the Source Servers page.

 click `Mark as Cutover Complete`.

Expected Result: The server status changes to Cutover Complete, signifying that the migration is finished. The cutover instance is now fully operational, and the source server can be archived (take `action` and select `mark as archived` )or decommissioned as needed.
