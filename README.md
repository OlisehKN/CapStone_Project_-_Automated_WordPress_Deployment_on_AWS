# <ins>Terraform CapStone Project - Automated WordPress Deployment on AWS</ins>

## <ins>Project Scenario</ins>

DigitalBoost, a digital marketing agency, aims to elevate its online presence by launching a high-performance WordPress website for their clients. As an AWS Solutions Architect, your task is to design and implement a scalable, secure, and cost-effective WordPress solution using various AWS services. Automation through Terraform will be key to achieving a streamlined and reproducible deployment process.

## <ins>Project Components</ins>

### 1.<ins>VPC Setup</ins>

<ins>Objective:</ins> Create a Virtual Privaate Cloud (VPC) to isolate and secure  the WordPress infrastructure.

<ins>Steps:</ins> 

  - Define IP address range for the VPC.

I compiled the following terraform code that states which provider, the region and the address range.

      # VPC
      
      provider "aws" {
        region = "us-east-1"
      }
      
      resource "aws_vpc" "main" {
        cidr_block = "10.0.0.0/16"
      }

  - Create VPC with public and private subnets.

I compiled the following terraform code that creates both private and public subnets

        # Public Subnets
        
        resource "aws_subnet" "public_1" {
          vpc_id                  = aws_vpc.main.id
          cidr_block              = "10.0.1.0/24"
          map_public_ip_on_launch = true
          availability_zone       = "us-east-1a"
        }
        
        resource "aws_subnet" "public_2" {
          vpc_id                  = aws_vpc.main.id
          cidr_block              = "10.0.2.0/24"
          map_public_ip_on_launch = true
          availability_zone       = "us-east-1b"
        }
        
        # Private Subnets
        
        resource "aws_subnet" "private_1" {
          vpc_id            = aws_vpc.main.id
          cidr_block        = "10.0.3.0/24"
          availability_zone = "us-east-1a"
        }
        
        resource "aws_subnet" "private_2" {
          vpc_id            = aws_vpc.main.id
          cidr_block        = "10.0.4.0/24"
          availability_zone = "us-east-1b"
        }


  - Configure route tables for each subnet.

I compiled the following terraform code that creates route tables for each subnet.

        # Public Route tables
        
        resource "aws_route_table" "public_rt" {
          vpc_id = aws_vpc.main.id
        
          route {
            cidr_block = "0.0.0.0/0"
            gateway_id = aws_internet_gateway.igw.id
          }
        }
        
        resource "aws_route_table_association" "public_1_assoc" {
          subnet_id      = aws_subnet.public_1.id
          route_table_id = aws_route_table.public_rt.id
        }

        resource "aws_route_table_association" "public_2_assoc" {
          subnet_id      = aws_subnet.public_2.id
          route_table_id = aws_route_table.public_rt.id
        }

        # Private Route tables
        
        resource "aws_route_table" "private_rt" {
          vpc_id = aws_vpc.main.id
        }
        
        resource "aws_route_table_association" "private_1_assoc" {
          subnet_id      = aws_subnet.private_1.id
          route_table_id = aws_route_table.private_rt.id
        }
        
        resource "aws_route_table_association" "private_2_assoc" {
          subnet_id      = aws_subnet.private_2.id
          route_table_id = aws_route_table.private_rt.id
        }

  <ins>Variables.tf</ins>

Below is a list of variables for the VPC terraform code

        variable "aws_region" {
          description = "AWS region"
          default     = "us-east-1"
        }
        
        variable "vpc_cidr" {
          description = "VPC CIDR block"
          default     = "10.0.0.0/16"
        }
        
        variable "public_subnet_1_cidr" {
          description = "CIDR block for public subnet 1"
          default     = "10.0.1.0/24"
        }
        
        variable "public_subnet_2_cidr" {
          description = "CIDR block for public subnet 2"
          default     = "10.0.2.0/24"
        }
        
        variable "private_subnet_1_cidr" {
          description = "CIDR block for private subnet 1"
          default     = "10.0.3.0/24"
        }
        
        variable "private_subnet_2_cidr" {
          description = "CIDR block for private subnet 2"
          default     = "10.0.4.0/24"
        }
        
        variable "az_1" {
          description = "Availability zone 1"
          default     = "us-east-1a"
        }
        
        variable "az_2" {
          description = "Availability zone 2"
          default     = "us-east-1b"
        }

  <ins>Outputs.tf</ins>

Below are a list of different Outputs

        output "vpc_id" {
          description = "The ID of the VPC"
          value       = aws_vpc.main.id
        }
        
        output "public_subnet_1_id" {
          description = "ID of public subnet 1"
          value       = aws_subnet.public_1.id
        }
        
        output "public_subnet_2_id" {
          description = "ID of public subnet 2"
          value       = aws_subnet.public_2.id
        }
        
        output "private_subnet_1_id" {
          description = "ID of private subnet 1"
          value       = aws_subnet.private_1.id
        }
        
        output "private_subnet_2_id" {
          description = "ID of private subnet 2"
          value       = aws_subnet.private_2.id
        }

  <Deploying the VPC</ins>

After inputing the code, i run the following command lines to initialize, review and apply the terraform code to the ec2 instance

  - Initialize Terraform

        terraform init

  - Review the resources to be created

        terraform plan

  - Apply the Configuration

        terraform apply

### 2.<ins>Public and Private Subnet with NAT Gateway</ins>

<ins>Objective:</ins> Implement a secure network architecture with public and private subnets. Use a NAT Gateway for private subnet internet access.

<ins>Steps:</ins> 

  - Set up a public subnet for resources accessible from the internet.

        resource "aws_subnet" "public" {
          vpc_id                  = aws_vpc.main.id
          cidr_block              = var.public_subnet_cidr
          map_public_ip_on_launch = true
          availability_zone       = var.az
        }

  - Create a private subnet for resources with no direct internet access.

        resource "aws_subnet" "private" {
          vpc_id            = aws_vpc.main.id
          cidr_block        = var.private_subnet_cidr
          availability_zone = var.az
        }

  - Configure a NAT Gateway for private subnet internet access.

        resource "aws_eip" "NGW 1" {
          domain = "vpc"
        }
        
        resource "aws_nat_gateway" "NGW 1" {
          allocation_id = aws_eip.nat.id
          subnet_id     = aws_subnet.public.id
        }

### 3.<ins> AWS MySQL RDS Setup</ins>

<ins>Objective:</ins> Deploy a managed MySQL database using Amazon RDS for WordPress data storage.

<ins>Steps:</ins>

  - Create an Amazon RDS instance with the MySQL engine.

  - Configure security groups for the RDS instance.

        # Amazon RDS Instance
    
        provider "aws" {
          region = var.aws_region
        }
        
        resource "aws_db_instance" "mysql" {
          allocated_storage    = var.allocated_storage
          engine              = "mysql"
          engine_version      = var.engine_version
          instance_class      = var.instance_class
          db_name             = var.db_name
          username           = var.db_username
          password           = var.db_password
          parameter_group_name = "default.mysql8.0"
          skip_final_snapshot = true
          publicly_accessible = false
          storage_encrypted   = true
          vpc_security_group_ids = [var.security_group_id]
          db_subnet_group_name   = var.db_subnet_group_name
        }

  <ins>Variables.tf</ins>

Below are the variables for the RDS intance.

      variable "aws_region" {
        description = "AWS region"
        default     = "us-east-1"
      }
      
      variable "allocated_storage" {
        description = "The allocated storage in gigabytes"
        default     = 20
      }
      
      variable "engine_version" {
        description = "The version of the database engine"
        default     = "8.0"
      }
      
      variable "instance_class" {
        description = "The instance type of the RDS instance"
        default     = "db.t3.micro"
      }
      
      variable "db_name" {
        description = "The name of the database"
        default     = "mydatabase"
      }
      
      variable "db_username" {
        description = "The master username for the database"
        default     = "admin"
      }
      
      variable "db_password" {
        description = "The master password for the database"
        default     = "changeme123"
        sensitive   = true
      }
      
      variable "security_group_id" {
        description = "The security group ID for the RDS instance"
      }
      
      variable "db_subnet_group_name" {
        description = "The DB subnet group name"
      }

  <ins>Outputs.tf</ins>

Below are the Outputs for the RDS Instance

      output "rds_endpoint" {
        description = "The connection endpoint for the RDS instance"
        value       = aws_db_instance.mysql.endpoint
      }
      
      output "rds_instance_id" {
        description = "The ID of the RDS instance"
        value       = aws_db_instance.mysql.id
      }

  - Connect WordPress to the RDS database.

            provider "aws" {
              region = var.aws_region
            }
            
            resource "aws_instance" "wordpress" {
              ami           = var.ami_id
              instance_type = var.instance_type
              subnet_id     = var.subnet_id
              security_groups = [var.security_group_id]
              key_name      = var.key_pair
            
              user_data = <<-EOF
                          #!/bin/bash
                          yum update -y
                          amazon-linux-extras enable php8.0
                          yum install -y httpd php php-mysqlnd
                          systemctl start httpd
                          systemctl enable httpd
            
                          cd /var/www/html
                          wget https://wordpress.org/latest.tar.gz
                          tar -xzf latest.tar.gz
                          cp -r wordpress/* .
                          rm -rf wordpress latest.tar.gz
            
                          cat <<EOT > wp-config.php
                          <?php
                          define('DB_NAME', '${var.db_name}');
                          define('DB_USER', '${var.db_username}');
                          define('DB_PASSWORD', '${var.db_password}');
                          define('DB_HOST', '${var.db_host}');
                          define('DB_CHARSET', 'utf8');
                          define('DB_COLLATE', '');
                          
                          \$table_prefix = 'wp_';
                          define('WP_DEBUG', false);
                          
                          if ( !defined('ABSPATH') ) {
                            define('ABSPATH', dirname(__FILE__) . '/');
                          }
                          require_once(ABSPATH . 'wp-settings.php');
                          ?>
                          EOT
            
                          chown -R apache:apache /var/www/html
                          chmod -R 755 /var/www/html
                          systemctl restart httpd
                          EOF
            
              tags = {
                Name = "WordPress-Instance"
              }
            }

  <ins>Variables.tf and Output.tf</ins>

Below are the variables and outputs for the RDS instance WordPress configuration.

      variable "aws_region" {
        description = "AWS region"
        default     = "us-east-1"
      }
      
      variable "ami_id" {
        description = "AMI ID for the WordPress instance"
        default     = "ami-0c55b159cbfafe1f0"
      }
      
      variable "instance_type" {
        description = "EC2 instance type"
        default     = "t2.micro"
      }
      
      variable "subnet_id" {
        description = "Subnet ID for the WordPress instance"
      }
      
      variable "security_group_id" {
        description = "Security group ID for the WordPress instance"
      }
      
      variable "key_pair" {
        description = "Key pair name for SSH access"
      }
      
      variable "db_name" {
        description = "Database name"
      }
      
      variable "db_username" {
        description = "Database username"
      }
      
      variable "db_password" {
        description = "Database password"
        sensitive   = true
      }
      
      variable "db_host" {
        description = "Database host (RDS endpoint)"
      }
      
      output "wordpress_public_ip" {
        description = "Public IP of the WordPress instance"
        value       = aws_instance.wordpress.public_ip
      }

### 4.<ins>EFS Setup for WordPress Files</ins>

<ins>Objectives:</ins> Utilize Amazon Elastic File System (EFS) to store WordPress files for scalable and shared access.

<ins>Steps:</ins> 

  - Create an EFS file system.

