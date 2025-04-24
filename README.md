# Project-3-MIGRATION-OF-BITBUCKET-DATA-CENTER-FROM-ONE-REGION-TO-ANOTHER-REGION-LEVERAGING-MGN-IN-AWS

## MIGRATION-OF-BITBUCKET-DATA-from-one-region(N Virginia)-to-another(Oregon)
## Prerequisites
AWS Account: Ensure you have access to the AWS accounts for both source and target regions.
Bitbucket Data Center: Ensure you have administrative access to your existing Bitbucket Data Center.
AWS Application Migration Service (MGN): Verify that MGN is enabled in both the source and target regions.
Networking: Ensure proper VPC, subnets, security groups, and routing are configured in both regions to support Bitbucket’s requirements.
Backups: Create a full backup of your Bitbucket Data Center (database, shared storage, and configuration files) before migration.

## Architecture

- Source Region (us-east-1):- EC2 instances hosting the Bitbucket application and data.

- Target Region (us-west-2):- Replicated EC2 instances for hosting Bitbucket data after migration.

- AWS MGN:- Automates replication and server launch.

- DNS Update:- Redirect traffic to servers in Oregon.
- 
![image](https://github.com/user-attachments/assets/75c714aa-379a-4a89-b68e-c7bb2581e1ee)

## Bitbucket Data Center in AWS:

Log in to the AWS Management Console and launch an instance. use t2medium, linux 2, 50GB, keypair,enable auto-assign public IP, open port 22(ssh), 7990(bitbucket), take note of the region.

<img width="899" alt="Image" src="https://github.com/user-attachments/assets/8a180850-558d-4f95-89d2-af899ab33f3a" />

## user data
The user data will update the system, install docker, create a directory for bitbucket data, and create the bitbucket docker container on the server.
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


- run 
```
docker ps #to see the containers.
```
<img width="467" alt="Image" src="https://github.com/user-attachments/assets/b85daaf8-27fb-4164-8462-8f9677573a70" />

- run
- Check if docker is actice and running 
```
sudo systemctl status docker
```
<img width="482" alt="Image" src="https://github.com/user-attachments/assets/261fc806-326e-4fea-8dd5-b358e9c5bd08" />

- copy the public ip address of the instance and add the port number e.g `52.7.182.131:7990`
![Image](https://github.com/user-attachments/assets/f4f49d03-be5e-40ed-96d2-7276105e32af)

# Perform the Migration Using MGN
## Start Replication:
In the MGN console, start the replication process.
AWS MGN will create staging resources in the target region to replicate the source instances.

# intallation of application agent
- use aws-mgn application migration service in the target region (Oregon)
- Navigate to the AWS Application Migration Service (MGN) section.
-In the Source Servers section, click `Add Source Server`.
- Download the Replication Agent installer for your operating system
- select `Linux` or `windows`
- add `the access key` and `secret key` gotten from `IAM` `user` `access key` from aws.
- 
<img width="904" alt="Image" src="https://github.com/user-attachments/assets/035d14e8-e41b-43fb-b19f-d67f05cfe407" />

-Terminal

  - Download the installer using this command on you source terminal
 
   ```
  sudo wget -O ./aws-replication-installer-init https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
   ```

 - Make the installer executable and Run the installer with root privileges using this command, make sure Replace <AWS_REGION> with the region where AWS MGN is configured (e.g., us-west-2 target region)

  -  Install the application agent
  ```
  sudo chmod +x aws-replication-installer-init; sudo ./aws-replication-installer-init --region us-west-2 # Target region Oregon
  ```
  <img width="713" alt="Image" src="https://github.com/user-attachments/assets/fb40e040-9c63-4fc4-98ec-f0ed16c32866" />
  

- Verify Installation
After installation, the replication agent will register the source server with AWS MGN.
-Check the AWS MGN Console in the Source Servers section:
The source server should appear as "Pending" or "Ready for replication."

-select `Back`
-select `source server`

- monitor the replication as it moves from `Not ready to ready to test, test in progress, ready for cutover, cutover in progress, cutover complete`
- The replication intiation step will have this created:
-  Create security groups, Launch Replication Server, Boot Replication Server, Authenticate with service, Download replication software, Create staging disks, Attach staging disks, Pair Replication Server with AWS Replication Agent, Connect AWS Replication Agent to Replication Server and Start data transfer
-  
![Image](https://github.com/user-attachments/assets/317e5a40-78d2-4941-bcbc-9c0bcd84aeb3)

- Ready for testing.
  - Click `test and cutover` then select `launch test instance`
  - An IP internal main testing server is created, you can test this instance in the internet.
  - if it does not have a public address, create one using `Elastic IP`

        
- test in progress
  - creat an Elastic IP address
  - Go to the ec2 internal instance and select it
  - select `elastic Ip` at the left
  - click `Allocate elastic ip address`
  - select the allocated Ip and take `associate IP`
  - chose the `instance` and the `private ip`
  - also attached a `security group` to access it openning the right ports e.g `7990`
    <img width="907" alt="Image" src="https://github.com/user-attachments/assets/0744e486-ab8e-4e40-ac13-f0346e240a20" />
    <img width="955" alt="Image" src="https://github.com/user-attachments/assets/9307de85-9a6a-4121-8952-e7d6bbe5ba0c" />

    - An IP internal main testing server is created, you can test this instance in the internet.
     ![Image](https://github.com/user-attachments/assets/183a3492-4dc8-4be8-af16-801323ea05a7)
    - This shows that we have migrated to the server.
 
 - source server
  - click  `test and cutover` and select `mark as ready for cutover` then take continue. This is do the final migration, will create a new server for your actual migration. the cutover will delete a replication server.

    <img width="884" alt="Image" src="https://github.com/user-attachments/assets/ad0634e4-179d-4020-85c2-db3a77e056e2" />

    - The replication instances are terminated
      <img width="928" alt="Image" src="https://github.com/user-attachments/assets/ad2cf711-ffaa-4429-ad1c-3277da70279e" />
 
  - Perform a Test Cutover
  - select test and cutover and select launch cutover instance and select launch; this will create another server to replicate the serve.

 <img width="884" alt="Image" src="https://github.com/user-attachments/assets/82585c51-1bf2-478e-a065-d9d997c3da15" />
 
- cutover in progress 
  - On the Source Servers page, select the server marked Ready for Cutover.

- Click Launch Cutover Instances.

- Review and confirm the instance settings, then click Launch.

Expected Result: The cutover instance is launched in AWS, and the server enters the Cutover in Progress state. This step involves transitioning production traffic to the new environment.
<img width="887" alt="Image" src="https://github.com/user-attachments/assets/f30b539f-95c6-4573-b569-23a7c4dabcb9" />
 - New servers (IP internal, webapp-server, AWS application Migration Service and Replication server) are created with with all the data inside
   <img width="928" alt="Image" src="https://github.com/user-attachments/assets/f959b95a-c1ad-4513-9221-6a56238ed7f8" />
   - Test the newly Internal server by associating the elastic ip and security group before testing on the server.
   
- Cutover Complete
  - After verifying that the cutover instance is operating correctly, return to the Source Servers page.
  - click `test and cutover` `finalize cutover` # all the servers involved in the migration and replication data will be deleted acept you last application server. 
 click `Mark as Cutover Complete`.

Expected Result: The server status changes to Cutover Complete, signifying that the migration is finished. The cutover instance is now fully operational, and the source server can be archived (take `action` and select `mark as archived` ) or decommissioned as needed.
- migration successful to the new region Oregon us-west-2

<img width="908" alt="Image" src="https://github.com/user-attachments/assets/f2b11b86-192c-44ea-bb4d-fd02a3045fc8" />

## Conclusion
This project successfully outlines the migration of Bitbucket data from the AWS N Virginia region to the Oregon region using AWS Application Migration Service (MGN). By following the AWS Console procedures, the migration ensures minimal downtime and preserves data integrity. This process is ideal for organizations looking to transition workloads across AWS regions.



