[section:overview]
### Overview

The goal of this project is to build a **centralized networking architecture** using an **AWS Transit Gateway (TGW)** that connects multiple VPCs securely and efficiently.

In multi-VPC architectures, each VPC typically connects via peering. As the number of VPCs grows, managing peer connections becomes complex.  
The AWS Transit Gateway simplifies this by acting as a **hub** for inter-VPC and hybrid connectivity.

[section:architecture Overview]

### Architecture Overview

**Diagram:**

<img width="753" height="723" alt="image" src="https://github.com/user-attachments/assets/855bcc4e-9338-4887-b07f-0a43ebd57102" />

**Components:**
- Transit Gateway
- Three VPCs
- TGW attachment
- route tables and associations
- HCL remote backend

[section:project structre]
### Project structure
- main.tf
- variables.tf
- outputs.tf
- vpc.tf
- tgw.tf
- routes.tf
- provider.tf
- backend.tf

So Each `.tf` file represents a logical part of the infrastrucre.

[section:implementation Steps]
### Implementation Steps

**provider**

In `provider.tf`
```
provider "aws" {
  region     = var.region
  
}
```
In `variables.tf`
```
# AWS Region
variable "region" {
  description = "The AWS region to deploy resources"
  default     = "us-east-1"
}

# VPC CIDR Blocks
variable "vpc_1_cidr" {
  description = "CIDR block for VPC 1"
  default     = "10.0.0.0/16"
}

variable "vpc_2_cidr" {
  description = "CIDR block for VPC 2"
  default     = "11.0.0.0/16"
}

variable "vpc_3_cidr" {
  description = "CIDR block for VPC 3"
  default     = "12.0.0.0/16"
}

# Subnet CIDR Blocks
variable "vpc_1_private_subnets" {
  description = "Private subnet CIDRs for VPC 1"
  default     = "10.0.0.0/24"
}

variable "vpc_2_private_subnets" {
  description = "Private subnet CIDRs for VPC 2"
  default     = "11.0.0.0/24"
}

variable "vpc_3_private_subnets" {
  description = "Private subnet CIDRs for VPC 3"
  default     = "12.0.1.0/24"
}

variable "vpc_3_public_subnets" {
  description = "Public subnet CIDRs for VPC 3"
  default     = "12.0.2.0/24"
}

# Tags
variable "tags" {
  description = "Tags for resources"
  default = {
    Environment = "terraformChamps"
    Owner       = "YoussefShawky"
  }
}

# EC2 AMI and Instance Type
variable "ami_id" {
  description = "AMI ID for EC2 instances"
  default     = "ami-0453ec754f44f9a4a" # Replace with your AMI ID
}

variable "instance_type" {
  description = "Instance type for EC2 instances"
  default     = "t2.micro"
}

#AVz for subnets

variable "availability_zone_vpc_1_private_subnet" {
  description = "Availability zone vpc 1 private subnet"
  default     = "use1-az1"
}

variable "availability_zone_vpc_2_private_subnet" {
  description = "Availability zone vpc 2 private subnet"
  default     = "use1-az2"
}

variable "availability_zone_vpc_3_public_subnet" {
  description = "Availability zone vpc 3 public subnet"
  default     = "use1-az3"
}

variable "availability_zone_vpc_3_private_subnet" {
  description = "Availability zone vpc 3 private subnet"
  default     = "use1-az3"
}


variable "env" {
  default = "test"
}
```
In `backend.tf`
```
terraform {
  backend "remote" {
    hostname     = "app.terraform.io"
    organization = "Youssef-shawky"
    workspaces {
      name = "transit_gateway_workspace"
    }
  }
}
```
In `vpc.tf`
```
#   VPC 1 with Private Subnet
resource "aws_vpc" "vpc_1" {
  cidr_block = var.vpc_1_cidr
  tags = {
    Name = "${var.env}-vpc-1"
  }
}

resource "aws_subnet" "vpc_1_private_subnet" {
  vpc_id               = aws_vpc.vpc_1.id
  cidr_block           = var.vpc_1_private_subnets
  availability_zone_id = var.availability_zone_vpc_1_private_subnet
  tags = {
    Name = "${var.env}-private-subnet-vpc-1"
  }
}

# ---- VPC 2 with Private Subnet ----
resource "aws_vpc" "vpc_2" {
  cidr_block = var.vpc_2_cidr
  tags = {
    Name = "${var.env}-vpc-2"
  }
}

resource "aws_subnet" "vpc_2_private_subnet" {
  vpc_id               = aws_vpc.vpc_2.id
  cidr_block           = var.vpc_2_private_subnets
  availability_zone_id = var.availability_zone_vpc_2_private_subnet
  tags = {
    Name = "${var.env}-private-subnet-vpc-2"
  }
}

# ---- VPC 3 with Public Subnet ----
resource "aws_vpc" "vpc_3" {
  cidr_block = var.vpc_3_cidr
  tags = {
    Name = "${var.env}-vpc-3"
  }
}

resource "aws_subnet" "vpc_3_private_subnet" {
  vpc_id               = aws_vpc.vpc_3.id
  cidr_block           = var.vpc_3_private_subnets
  availability_zone_id = var.availability_zone_vpc_3_private_subnet
  tags = {
    Name = "${var.env}-private-subnet-vpc-3"
  }
```
In `ec2.tf`
```
#  SSH keys configurations
resource "aws_key_pair" "my_key" {
  key_name   = "my-key"
  public_key = file("my-key.pub") # Path to the generated public key
}



# ---- EC2 Instances ----
resource "aws_instance" "vpc_1_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.vpc_1_private_subnet.id
  key_name      = aws_key_pair.my_key.key_name # Reference the key pair

  security_groups = [aws_security_group.private_1_ec2_sg.id]

  tags = var.tags
}

resource "aws_instance" "vpc_2_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.vpc_2_private_subnet.id
  key_name      = aws_key_pair.my_key.key_name # Reference the key pair

  security_groups = [aws_security_group.private_2_ec2_sg.id]

  tags = var.tags
}

resource "aws_instance" "vpc_3_public_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.vpc_3_public_subnet.id
  key_name      = aws_key_pair.my_key.key_name # Reference the key pair

  security_groups = [aws_security_group.public_ec2_sg.id]


  tags                        = var.tags
  associate_public_ip_address = true
}
```
In `sg.tf`
```
# Security Group for Public Subnet EC2
resource "aws_security_group" "public_ec2_sg" {
  vpc_id      = aws_vpc.vpc_3.id
  name        = "public-ec2-sg"
  description = "Allow SSH access to EC2 in public subnet"

  # Inbound rules
  ingress {
    description = "Allow SSH from anywhere (for testing)"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with your specific IP for better security


  }

  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]

  }

  tags = var.tags
}


# Security Group for private Subnets EC2
resource "aws_security_group" "private_1_ec2_sg" {
  vpc_id      = aws_vpc.vpc_1.id
  name        = "private-1-ec2-sg"
  description = "Allow SSH access to EC2 in public subnet"

  ingress {
    description = "Allow SSH from anywhere (for testing)"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with your specific IP for better security


  }

  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]

  }

  tags = var.tags

}

resource "aws_security_group" "private_2_ec2_sg" {
  vpc_id      = aws_vpc.vpc_2.id
  name        = "private-2-ec2-sg"
  description = "Allow SSH access to EC2 in public subnet"

  ingress {
    description = "Allow SSH from anywhere (for testing)"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with your specific IP for better security

  }

  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]

  }

  tags = var.tags

}
```
In `gateways.tf`
```
# ---- Internet Gateway for Public Subnet ----
resource "aws_internet_gateway" "vpc3_igw" {
  vpc_id = aws_vpc.vpc_3.id
  tags = {
    Name = "${var.env}-igw-vpc-3"
  }
}

#     Elastic IP and NAT Gateway for public Subnet 
resource "aws_eip" "vpc3_nat_eip" {
  tags = {
    Name = "${var.env}-eip-nat"
  }
}

#   NAT Gateway in public subnet
resource "aws_nat_gateway" "vpc3_nat" {
  allocation_id = aws_eip.vpc3_nat_eip.id
  subnet_id     = aws_subnet.vpc_3_public_subnet.id

  tags = {
    Name = "${var.env}-nat-vpc-3"
```
In `tgw.tf`
```
#     Transit Gateway 
resource "aws_ec2_transit_gateway" "tgw" {
  default_route_table_association = "disable"
  tags = {
    Name = "${var.env}-tgw"
  }
}

# TGW Route table
resource "aws_ec2_transit_gateway_route_table" "tgw_rt" {
  transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  tags = {
    Name = "${var.env}-tgw-rt"
  }
}

# Attach VPCs to Transit Gateway
resource "aws_ec2_transit_gateway_vpc_attachment" "vpc1_attachment" {
  transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  vpc_id             = aws_vpc.vpc_1.id
  subnet_ids         = [aws_subnet.vpc_1_private_subnet.id]
  tags = {
    Name = "${var.env}-tgw-attachment-vpc-1"
  }
}


resource "aws_ec2_transit_gateway_route" "route_to_vpc_1" {
  destination_cidr_block         = var.vpc_1_cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc1_attachment.id
}

resource "aws_ec2_transit_gateway_vpc_attachment" "vpc2_attachment" {
  transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  vpc_id             = aws_vpc.vpc_2.id
  subnet_ids         = [aws_subnet.vpc_2_private_subnet.id]
  tags = {
    Name = "${var.env}-tgw-attachment-vpc-2"
  }
}


resource "aws_ec2_transit_gateway_route" "route_to_vpc_2" {
  destination_cidr_block         = var.vpc_2_cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc2_attachment.id
}

resource "aws_ec2_transit_gateway_vpc_attachment" "vpc3_attachment" {
  transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  vpc_id             = aws_vpc.vpc_3.id
  subnet_ids         = [aws_subnet.vpc_3_private_subnet.id]
  tags = {
    Name = "${var.env}-tgw-attachment-vpc-3"
  }
}


resource "aws_ec2_transit_gateway_route" "route_to_vpc_3" {
  destination_cidr_block         = var.vpc_3_cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc3_attachment.id
}

resource "aws_ec2_transit_gateway_route" "route_to_igw" {
  destination_cidr_block         = "0.0.0.0/0"
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc3_attachment.id
}

# Associate the attachment to TGW
resource "aws_ec2_transit_gateway_route_table_association" "vpc1_attach_route_association" {

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc1_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id

}

resource "aws_ec2_transit_gateway_route_table_association" "vpc2_attach_route_association" {

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc2_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id

}

resource "aws_ec2_transit_gateway_route_table_association" "vpc3_attach_route_association" {

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc3_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id

}


# Enable Propagation from Each VPC to the Transit Gateway Route Table
resource "aws_ec2_transit_gateway_route_table_propagation" "vpc1_propagation" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc1_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "vpc2_propagation" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc2_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "vpc3_propagation" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.vpc3_attachment.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.tgw_rt.id
}
```
Finally, `routes.tf`
```

#  public subnet route table  
resource "aws_route_table" "vpc_3_public_route_table" {
  vpc_id = aws_vpc.vpc_3.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.vpc3_igw.id
  }

  route {
    cidr_block         = var.vpc_1_cidr
    transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  }

  route {
    cidr_block         = var.vpc_2_cidr
    transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  }




  tags = {
    Name = "${var.env}-rt-3-vpc-3"
  }
}
resource "aws_route_table_association" "associate_public_rt_to_subnet" {
  subnet_id      = aws_subnet.vpc_3_public_subnet.id
  route_table_id = aws_route_table.vpc_3_public_route_table.id
}


resource "aws_route_table" "vpc_3_private_route_table" {
  vpc_id = aws_vpc.vpc_3.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.vpc3_nat.id
  }

  tags = {
    Name = "centralized-private-rt"
  }
}

resource "aws_route_table_association" "association_private_rt_to_subnet" {
  subnet_id      = aws_subnet.vpc_3_private_subnet.id
  route_table_id = aws_route_table.vpc_3_private_route_table.id
}

#  Vpc_1 route table and associate
resource "aws_route_table" "vpc1_private_route_table" {
  vpc_id = aws_vpc.vpc_1.id

  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  }



  tags = {
    Name = "${var.env}-rt-1-vpc-1"
  }

}

resource "aws_route_table_association" "associate_route_table_1_to_subnet" {
  subnet_id      = aws_subnet.vpc_1_private_subnet.id
  route_table_id = aws_route_table.vpc1_private_route_table.id
}

#  Vpc_2 route table and associate
resource "aws_route_table" "vpc2_private_route_table" {
  vpc_id = aws_vpc.vpc_2.id

  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.tgw.id
  }


  tags = {
    Name = "${var.env}-rt-2-vpc-2"
  }

}

resource "aws_route_table_association" "associate_route_table_2_to_subnet" {
  subnet_id      = aws_subnet.vpc_2_private_subnet.id
  route_table_id = aws_route_table.vpc2_private_route_table.id
}
```

[section:circleci pipeline overview]

### CircleCI Pipeline Overview
CircleCI is acting as your Terraform automation pipeline.
It deos 3 key things:
- Runs Terraform Plan
- Requires Manual Approval
- Runs Terraform Apply

[section:How to Configure CircleCI]
### How to Configure CircleCI for this Project
1- Your CircleCI pipeline must live in:
```
  .your-repo/
 └── .circleci/
     └── config.yml
```
2- Connect Github Repo to CircleCI
- Go to https://app.circleci.com/
- Login with GitHub
- Choose your repository
- Click Set Up Project
  CircleCI automatically detects your config.yml.

3- Add Environment Variables in CircleCI
- Go to:
Project Settings → Environment Variables
Add the following variables:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION`
- `TF_API_TOKEN`

[section:pipeline implementation]

### Pipeline Implementation
1- **prepare Job**

- This job runs in a Docker container that already has Terraform installed.
- Pulls your Git repo (Terraform code).
```
jobs:
  prepare:
    docker:
      - image: hashicorp/terraform:1.9.4
    steps:
      - checkout
      - run:
          name: Prepare Workspace
          command: echo "Preparing workspace..."
```
2- **plan Job**

This job runs Terraform plan and saves the plan output for later.
- Creates Terraform credentials file
- Inserts the Terraform Cloud API token (`$TF_API_TOKEN`)
- This allows Terraform to authenticate with Terraform Cloud
  ```
  plan:
    docker:
      - image: hashicorp/terraform:1.9.4
    steps:
      - checkout
      - run:
          name: Set Terraform Credentials
          command: |
            mkdir -p ~/.terraform.d
            echo '{"credentials":{"app.terraform.io":{"token":"'$TF_API_TOKEN'"}}}' > ~/.terraform.d/credentials.tfrc.json
  ```
  3- **Terrafrom Init**

  - Downloads providers
  - Connects to Terraform Cloud backend
  - Configures remote state backend
  - Makes AWS credentials available to Terraform providers
    ```
     - run:
          name: Terraform Init
          command: terraform init -upgrade
      - run:
          name: Set up AWS Credentials
          command: |
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION
    ```
    4- ** Terrafrom Plan**

    - Creates a deployment plan
    - Saves the plan to a file (`tfplan_tgw`)
      ```
       - run:    
          name: Terraform Plan
          command: terraform plan -out=tfplan_tgw
      ```
      5- ** Store Aritfacts in workspace**

      This saves important Terraform files so the next job can use them
      ```
      - persist_to_workspace:
          root: .
          paths:
            - .terraform
            - tfplan_tgw
            - .terraform.lock.hcl
      ```
      6- ** Approval before Apply**

      ```
      workflows:
         version: 2
         deploy:
           jobs:
             - prepare
             - plan
             - approval:  # Add manual approval step
                 type: approval
                 requires:
                   - plan
             - apply:
                 requires:
                   - approval
       ```
  
  [section: pipeline end]
  
  At The end of pipeline this ensures:
  - No accidental deployments
  - Immutable infrastructure deployment (plan → manual approval → apply)
  - Consistency between planned and applied resources
  - Secure authentication (AWS + Terraform Cloud)

  [section:full architecture]

   <img width="880" height="370" alt="image" src="https://github.com/user-attachments/assets/c06fd365-0bbc-4765-a3ec-a365a4ebc2d1" />
