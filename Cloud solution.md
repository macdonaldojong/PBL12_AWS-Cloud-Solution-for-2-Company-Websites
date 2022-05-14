## AWS Cloud Solution for 2 Company Websites Using A Reverse Proxy Technology

---

WARNING: This infrastructure set up is NOT covered by AWS free tier. Therefore, ensure to DELETE ALL the resources created immediately after finishing the project. Monthly cost may be shockingly high if resources are not deleted.

We will build a secure infrastructure inside AWS VPC (Virtual Private Cloud) network for a fictitious company that uses WordPress CMS for its main business website, and a [Tooling Website](https://github.com/stwalez/tooling) for their DevOps team. As part of the company’s desire for improved security and performance, a decision has been made to use a reverse proxy technology from NGINX to achieve this

Cost, Security, and Scalability are the major requirements for this project. Hence, implementing the architecture designed below, ensure that infrastructure for both websites, WordPress and Tooling, is resilient to Web Server’s failures, can accomodate to increased traffic and, at the same time, has reasonable cost.

![tooling_project_15](https://user-images.githubusercontent.com/41236641/166642557-b134d249-5e77-4be9-8252-887d73817e67.png)


### Domain Setup

Set up a web domain

Migrate DNS

- Set up a hostedzone to migrate the dns for the domain to route53
 ![Project15r53](https://user-images.githubusercontent.com/41236641/166643374-863eef71-6f64-4e52-8a85-fb7c5361f566.PNG)


Request a Certificate from ACM

- Navigate to AWS Certificate Manager. 
- Click on Request public certificate
- Provide the Fully Qualified Domain Name: *.onelovetooling.tk
- Select DNS Validation
- Navigate to Route53 and create the CNAME record provided:
- As soon as the DNS record propagates, refresh the ACM WebPage to verify that the certificate has been issued:
 ![Project53certificate](https://user-images.githubusercontent.com/41236641/166643710-df776fa8-09e0-425d-8896-be1a3affba8c.PNG)

Validate the DNS for ACM.

### Network Setup

Create VPC
- Create the vpc using any of the VPC Wizard. 
- This helps to create the VPC, subnets and route tables with less modifications. 
- Ensure two Availability zones, 2 Public and 4 Private subnets are selected
 ![Project15vpc](https://user-images.githubusercontent.com/41236641/166644060-4afcf1f8-365d-4598-aa8c-7956cd8201ed.PNG)
 ![Project15IG](https://user-images.githubusercontent.com/41236641/166644220-45cdff9c-5b92-411c-b389-a94f0493f42c.PNG)
![Project15subnet](https://user-images.githubusercontent.com/41236641/166644450-283f90f9-a8f8-4499-9cd1-209a97bdae19.PNG)


- Create Nat Gateway in the public subnet
![Project15nat](https://user-images.githubusercontent.com/41236641/166644591-63bab1be-2b25-46ff-aec1-205ef499532b.PNG)

- Cleanup the pre-configured routetables and configure only two route tables - private and public:
  ![](screenshots/route_table.png)

- Set the private routetable to point to NAT gateway for the four private subnets
  ![](screenshots/rtb_private.png)

- Set the public routetable to point to internet for the two public subnets
 ![Project15rt](https://user-images.githubusercontent.com/41236641/166644790-9b5e1142-da13-4fe6-b2d5-b1c2da312899.PNG)


Set up the Security Groups:
- Nginx Servers: 
  - Access to Nginx should only be allowed from a Application Load balancer (ALB) and Bastion
   ![Project15reverseproxy](https://user-images.githubusercontent.com/41236641/166645237-48e25082-a08d-4808-842f-038935a00182.PNG)


- Bastion Servers: 
  - Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. 
  - AWS has a feature while configuring Security Groups that detects your current IP (My-IP) in the security group
   ![Project15bastion](https://user-images.githubusercontent.com/41236641/166645438-2ec02577-e583-459c-b6e9-aa97a13930d1.PNG)

- Application Load Balancer: 
  - ALB will be available from the Internet
  - This SG will be used for both External and Internal ALB.
   ![Project15ALB](https://user-images.githubusercontent.com/41236641/166645580-8163cd38-c9c6-49c0-be0f-71f25af17764.PNG)

- Webservers: 
  - Access to Webservers should only be allowed from the Nginx servers, Internal ALB and Bastion servers
  ![Project15webserver](https://user-images.githubusercontent.com/41236641/166645814-1b894723-fa62-4b8c-8b9c-1522189a9d43.PNG)


- Data Layer: 
  - Access to the Data layer
  - Webservers should be able to connect to RDS
  - Nginx and Webservers will have access to EFS Mountpoint.
    ![image](https://user-images.githubusercontent.com/41236641/166646027-24a44eaf-a82a-49e4-91d8-0722ef551140.png)


### DataLayer Setup

Create Elastic File System EFS
- Ensure datalayer Security Group was selected
  ![](screenshots/efs_network.png)
- Create Access points for  Nginx server, Wordpress and Tooling server 
- Ensure the POSIX user and Creation info (0755) to allow connectivity and seamless administration of files.
  ![](screenshots/efs_ap.png)

Create KMS
- Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.
  ![](screenshots/kms.png)


Create RDS
- Create Database Subnet group using the private subnets 3 & 4
  ![](screenshots/rds_subnet_grp.png)


- Create Database, select free tier was used
- Ensure the datalayer SG and DB Subnet group was selected
 ![Project15datalayer](https://user-images.githubusercontent.com/41236641/166826549-2b0187ec-0af8-4c34-818e-17d380f0a80a.PNG)

- Ensure the KMS created previously was applied in the setup:
  ![Project15kms](https://user-images.githubusercontent.com/41236641/166826575-0635c87e-2017-4c6c-a204-89c560aae404.PNG)
- Note down the db credentials

### AMI Setup
- Launch a CentOS Instance and pre-install it with defaults requirements
  ```
  sudo su
  yum update -y
  yum install -y python3 ntp net-tools vim wget telnet epel-release htop
  ```

- Navigate to ```Instance > Actions > Images and Templates > Create Image ``` to create the AMI for the instances
 ![Project15ami](https://user-images.githubusercontent.com/41236641/166826476-c10c48a9-8d88-4571-b0b0-bd9b5c177246.PNG)


### Templates Setup for Bastion and Webservers

- Navigate to the EC2 Instances > Launch Templates to create launch templates using the Centos AMI. 
- Each pair of servers has unique userdatas and attributes.

- Bastion Launch Template
  - Create the Launch template for the Bastion server using the Centos AMI
  - Ensure that a public subnet is selected for this and other configurations like Instance Type, EBS Volume size, Bastion SG is configured.
  - Navigate to the Advanced settings > Userdata and paste the following to setup the Bastion server:

  ```
  #!/bin/bash
  hostnamectl set-hostname bastion-server
  yum update -y
  yum install -y git ansible
  ```
  - Launch an Instance from the Launch Template and verify that all configurations are reflected


WordPress Launch Template
  - Create the Launch template for the wordpress server using the Centos AMI
  - Ensure that a private subnet(1 or 2) is selected for this and other configurations like Instance Type, EBS Volume size, Webservers SG is configured.
  - Ensure that Public IP is removed from the network configuration as access to the wordpress instance will be via the Bastion server
  - Navigate to the Advanced settings > Userdata and paste the following to setup the Wordpress:

    ```
    #!/bin/bash
    hostnamectl set-hostname wordpress-server
    yum -y install git rpm-build make
    git clone https://github.com/aws/efs-utils
    cd efs-utils
    make rpm
    yum -y install build/amazon-efs-utils*rpm
    sed -i "s/stunnel_check_cert_hostname = true/stunnel_check_cert_hostname = false/g" /etc/amazon/efs/efs-utils.conf
    mkdir /var/www/
    mount -t efs -o tls,accesspoint=fsap-0c4988a080658a1ce fs-018d478f69b0acdf9:/ /var/www/
    yum install -y httpd 
    systemctl start httpd
    systemctl enable httpd
    cd ~
    echo "installing php"
    wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm       
    wget https://rpms.remirepo.net/enterprise/remi-release-7.rpm                      
    rpm -Uvh remi-release-7.rpm epel-release-latest-7.noarch.rpm                      
    yum-config-manager --enable remi-php74                         
    yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json                   
    sudo systemctl start php-fpm                                                      
    sudo systemctl enable php-fpm
    echo "installing wordpress"                                                     
    wget http://wordpress.org/latest.tar.gz
    tar xzvf latest.tar.gz
    rm -rf latest.tar.gz
    cp wordpress/wp-config-sample.php wordpress/wp-config.php
    mkdir /var/www/html/
    cp -R wordpress/* /var/www/html/
    cd /var/www/html/
    touch healthstatus
    sed -i "s/localhost/database-1.cvo1k7yvqffy.us-east-1.rds.amazonaws.com/g" wp-config.php 
    sed -i "s/username_here/admin/g" wp-config.php 
    sed -i "s/password_here/XAjXCswa7Lmnac/g" wp-config.php 
    sed -i "s/database_name_here/test/g" wp-config.php
    echo "setting selinux permissions"
    setsebool -P httpd_use_nfs 1
    setsebool -P httpd_can_network_connect=1
    setsebool -P httpd_execmem=1
    yum install -y mod_ssl
    systemctl restart httpd
    ```

- The WordPress userdata does the following:
  - Installs the amazon efs utility to connect to efs
  - removes tls limitation for efs which is specific to CentOS 7
  - Sets the selinux to allow the webserver access and utilize nfs
  - Creates a directory for the webserver files and mounts the efs access point partition to it
  - Install and enable apache webserver
  - Install and enable PHP
  - Create the /var/www/html directory
  - Downloads Wordpress
  - Move Wordpress content to the mounted partition
  - Replace inline the database credentials
  - Creates the health status check in the /var/www/html directory
  - Change the SELinux permissions to allow httpd use nfs, connect to db
  - Install ssl module on apache to activate https(443) listening port
  - Restart Apache to reflect the new configurations

 - Launch an Instance from the Launch Template and verify that all configurations are reflected

Tooling Launch Template
  - Create the Launch template for the nginx server using the Centos AMI
  - Ensure that a public subnet is selected for this and other configurations like Instance Type, EBS Volume size is configured.
  - Navigate to the Advanced settings > Userdata and paste the following to setup the Nginx:

    ```
    #!/bin/bash
    hostnamectl set-hostname tooling-server
    yum -y install git rpm-build make
    git clone https://github.com/aws/efs-utils
    cd efs-utils
    make rpm
    yum -y install build/amazon-efs-utils*rpm
    sed -i "s/stunnel_check_cert_hostname = true/stunnel_check_cert_hostname = false/g" /etc/amazon/efs/efs-utils.conf
    mkdir /var/www/
    sudo mount -t efs -o tls,accesspoint=fsap-0e1b08acb7e4323ee fs-018d478f69b0acdf9:/ /var/www/
    cd ~
    yum install -y httpd 
    systemctl start httpd
    systemctl enable httpd
    wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm       
    wget https://rpms.remirepo.net/enterprise/remi-release-7.rpm                      
    rpm -Uvh remi-release-7.rpm epel-release-latest-7.noarch.rpm                      
    yum-config-manager --enable remi-php74                         
    yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json                   
    sudo systemctl start php-fpm                                                      
    sudo systemctl enable php-fpm
    git clone https://github.com/stwalez/tooling.git
    mkdir /var/www/html
    cp -R tooling/html/*  /var/www/html/
    cd tooling
    mysql -h database-1.cvo1k7yvqffy.us-east-1.rds.amazonaws.com -u admin -pXAjXCswa7Lmnac -e "create database toolingdb"
    mysql -h database-1.cvo1k7yvqffy.us-east-1.rds.amazonaws.com -u admin -pXAjXCswa7Lmnac toolingdb < tooling-db.sql
    cd /var/www/html/
    touch healthstatus
    sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'admin', 'tooling');/$db = mysqli_connect('database-1.cvo1k7yvqffy.us-east-1.rds.amazonaws.com', 'admin', 'XAjXCswa7Lmnac', 'toolingdb');/g" functions.php
    setsebool -P httpd_use_nfs 1
    setsebool -P httpd_can_network_connect=1
    setsebool -P httpd_execmem=1
    yum install -y mod_ssl
    systemctl restart httpd

    ```
- The Tooling userdata does the following:
  - Installs the amazon efs utility to connect to efs
  - removes tls limitation for efs which is specific to CentOS 7
  - Sets the selinux to allow the webserver access and utilize nfs
  - Creates a directory for the webserver files and mounts the efs access point partition to it
  - Install and enable apache webserver
  - Install and enable PHP
  - Create the /var/www/html directory
  - Clones the tooling github repo
  - Move the tooling repo html ontent to the mounted partition
  - Creates the sql database 
  - Creates the sql data in the tooling.sql file
  - Creates the health status check in the /var/www/html directory
  - Replace inline the database credentials
  - Change the SELinux permissions to allow httpd use nfs, connect to db
  - Install ssl module on apache to activate https(443) listening port
  - Restart Apache to reflect the new configurations

- Launch an Instance from the Launch Template and verify that all configurations are reflected


Create a Target Group for Wordpress, Bastion and Tooling.
 - Ensure the TCP protocol is 22 for Bastion and HTTPS protocol 443 for WordPress and Tooling
 - Set the healthcheck path as "/healthstatus"
 - Register the already created instance
 - Ensure that the instances return healthy
   ![](screenshots/target_group_healthchecks.png)

Create an Automatic Scaling group and select desired capacity the matching launch template and target groups for the Wordpress, Bastion and Tooling.

### Create Internal Load Balancer

Navigate to the Load balancers and create an Internal Load Balancer 
  - Select the internal load balancer option 
  - Select IPv4
  - Map the load balancer to the two Availability zones
  - Select the ALB Security Group
  - select the wordpress target group
  - select listener port https
  - Select certificate issued earlier from ACM
   ![Project15ALB](https://user-images.githubusercontent.com/41236641/166826290-0106a59f-ec85-4f6e-bb59-9128abd235e1.PNG)


Configure the Internal Load Balancer 
  - Navigate to Listeners
  - Click on View/Edit rules for the target groups
  - Click on the Plus (+) icon to add a new rule.
  - Add the host header rule to map host to the specific target group, such that tooling.domain.com links to the tooling target groups and every other rule leads to the wordpress target groups



### Template Setup for NGINX
Setup Nginx Launch Template
- Create the Launch template for the nginx server using the Centos AMI
- Ensure that a public subnet is selected for this and other configurations like Instance Type, EBS Volume size is configured.
- Navigate to the Advanced settings > Userdata and paste the following to setup the Nginx:

  ```
  #!/bin/bash
  hostnamectl set-hostname nginx-rp-server
  yum -y install nginx git rpm-build make
  systemctl enable --now nginx
  git clone https://github.com/aws/efs-utils
  cd efs-utils
  make rpm
  yum -y install build/amazon-efs-utils*rpm
  sed -i "s/stunnel_check_cert_hostname = true/stunnel_check_cert_hostname = false/g" /etc/amazon/efs/efs-utils.conf
  mount -t efs -o tls,accesspoint=fsap-06e1daa069b75e66d fs-018d478f69b0acdf9:/ /var/log/nginx/
  setsebool -P httpd_use_nfs 1
  cd ~
  git clone https://github.com/stwalez/acs-config.git
  mkdir -p /etc/ssl/private
  chmod 700 /etc/ssl/private
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ACS.key -out /etc/ssl/certs/ACS.crt -config acs-config/req.conf
  mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
  mv acs-config/nginx.conf /etc/nginx/nginx.conf
  chcon -t httpd_config_t /etc/nginx/nginx.conf
  mkdir -p /var/www/html             
  touch /var/www/html/healthstatus
  systemctl restart nginx
  rm -rf acs-config
  ```

- The userdata does the following:
  - Installs the amazon efs utility to connect to efs
  - removes tls limitation which is specific to CentOS 7
  - Sets the selinux to allow the webserver access and utilize nfs
  - Clones a github repo to retrieve the preconfigured self signed ssl requests configuration file
  - Creates a required directory to input the self signed key
  - Creates a self signed key with the configuration file
  - Modifies the the nginx.conf to include setting for the reverse proxy configuration
  - Change the SELinux content of the new nginx.conf file
  - Create a new directory on the /var/www/html and inserts the healthstatus check page

- The specific nginx.conf file that was modified is shown below:
  ```
    server {
          listen       80;
          listen       443 http2 ssl;
          listen       [::]:443 http2 ssl;
          root         /var/www/html;
          server_name  *.onelovetooling.tk;      
          
          ssl_certificate /etc/ssl/certs/ACS.crt;
          ssl_certificate_key /etc/ssl/private/ACS.key;


          location /healthstatus {
          access_log off;
          return 200;
      }
      

        location / {
              proxy_set_header             Host $host;
              proxy_pass                   https://internal-onetooling-webservers-alb-1504327301.us-east-1.elb.amazonaws.com; 
          }
      }
  ```
- The nginx.conf includes a listening port for 443 as well as the path to the SSL Certificate.
- It also includes the proxy_pass to the Internal ALB. 
- This is such that all traffic directed from the external ALB to the nginx servers will be proxied to the internal ALB.
- Launch an Instance from the Launch Template and verify that all configurations are reflected

- All launch templates should be as shown: 
  ![LaunchTemplate](https://user-images.githubusercontent.com/41236641/166826127-7b7e5ee3-5181-4c22-9ae7-4933c78807a3.PNG)


- Create a Target Group for Nginx.
  - Ensure HTTPS protocol 443 for Nginx
  - Set the healthcheck path as "/healthstatus"
  - Register the already created instance
  - Ensure that the instances return healthy

- All target groups should be as shown:
  ![targetgroup](https://user-images.githubusercontent.com/41236641/166826086-de595a6b-1a62-45f3-a4c3-70a7e3e89514.PNG)


- Create an Automatic Scaling group and select desired capacity the matching launch template and target groups for Nginx.

- All ASGs should be as shown below:
![P15Autoscalinggroup](https://user-images.githubusercontent.com/41236641/166826025-1d56fc29-dade-4daa-8295-84cd2abb7640.PNG)



### Setup External LoadBalancer
- Create an external loadbalancer
  - Ensure "internet facing" is selected
  - Select the internal load balancer option 
  - Select IPV4
  - Map the load balancer to the two Availability zones
  - Select the ALB Security Group
  - select the nginx target group
  - select listener port https
  - Select certificate issued earlierfrom ACM

- Configure Route 53 and set routes to the External Load Balancer
  - Configure routes to tooling.<your_domain_name>.com as an alias to the External Load balancer, in this case, tooling.timiphoto.tk
  - Configure routes to wordpress.<your_domain_name>.com as an alias to the External Load balancer, in this case, wordpress.timiphoto.tk
![Project15domain](https://user-images.githubusercontent.com/41236641/166825943-3a46a886-260a-4beb-b1ce-bf1e30460852.PNG)
![Project15wordpress](https://user-images.githubusercontent.com/41236641/166826857-716f6e4d-d7f8-4c41-88c5-c7cc1d33fd14.PNG)
![Project15M](https://user-images.githubusercontent.com/41236641/166826915-68332072-01e3-42ab-863e-209bb1a3e52d.PNG)


### Test Connectivity
- Navigate to the tooling website by prefixing the domain with tooling and confirm that the page loads and is accessible:
- Navigate to the wordpress website with just the domain name,complete  and confirm that the page loads and is accessible:
 ![Project15tooling](https://user-images.githubusercontent.com/41236641/166825655-2ae1a3cc-4f8f-4237-81a3-cf6f5270e9d5.PNG)
