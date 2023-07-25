# Create an HTTPS Listener for the Application Load Balancer
 EC2 > Load Balancing [Loadbalancer] > select[Listeners-tab] > add listener 
 - protocol[https]
 - default action[forward]
 - target group -> select your TG[dev-tg]
 - Default SSL Cert -> From ACM -> Select you ssl cert [chia.click] -> [ADD]

- you have successfully create a https listener > view listener 

- For our http listener we need to change the default action to redirect traffic to the https listener Listener -> select [http] -> edit
     -  default action[remove] -> default action -> select[redirect]-> protocol[https] -> port [443]-> Save Changes

# SSH TO ONE OF YOUR PRIVATE WEB SERVER AZ1

```
sudo su
vi /var/www/html/wp-config.php

```


# PASTE THE CODE TO THIS PATH : vi /var/www/html/wp-config.php
- it makes sure your domain name is SECURE
- after you save the code
- so we have to update it in wordpress settings. 
- copy domain name -> /wp-admin -> Enter -> Login
- username: chia
- passwd: chia@123
- Settings > General > Update the wordpress URL and site address URL > From HTTP TO HTTPS ->
  remove the forward slash at the end -> SAVE CHANGES

  LOGIN AGAIN
  [PainChamp]



```
/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if(isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
```

# ASG : Create an Auto Scaling Group in AWS
- Using ASG to dynamically create our EC2 instances to make our website, highly available, scalable, fault tolerant and elastic.

1. Terminate the ec2 instances create before[websrv az1 and AZ2]
2. Create ASG but first we have to create a launch template
3. Instances -> LaunchTemplates :
   -  Create Launch template[dev-LT]
   -  autoscaling guidances -> select[provide..]
   -  select ami[amazonlinux2]
   -   instance type[t2.micro]
   -   select existing SG[websrv-SG]
Advance details -> user data [copy/paste-command] -> create launch templates
IN: for the userdata, make sure to edit the EFS info [JUST THE RED]

4. Autoscaling > Autoscaling Groups > Create ASG
  - name[dev-ASG] -> select Launch template[dev-LT] -> Next
  - network[dev-vpc] -> subnet -> select both subnet [private-app-subnet-AZ1/AZ2] -> Next
    
- Loadbalancing -> attach to an existing LB-> choose from LB target groups -> select TG[dev-TG] >
Health checks -> select[ELB]
Monitoring -> Enable[CloudWatch] -> NEXT
Group size[dc=2,min=1,max=4] -> 
Add notification -> SNS topic[select-topic???] 
Add Tag -> key[name] -> value[webSRV-ASG] -> Next -> Review > Create ASG

  EC2 > Target Groups > dev-tg
  Targets > Check if it is healthy
 -  MN: If the "Health Status" of the Instance is "unhealthy," wait for about 3-5 minutes to allow all the commands in the "user data" to run and install the software.
- After waiting 3-5 minutes, click the refresh button, and the instances should be healthy.


# USERDATA

```
#!/bin/bash
yum update -y
sudo yum install -y httpd httpd-tools mod_ssl
sudo systemctl enable httpd 
sudo systemctl start httpd
sudo amazon-linux-extras enable php7.4
sudo yum clean metadata
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld
echo "fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a
chown apache:apache -R /var/www/html
sudo service httpd restart
```
