![diagram](Terraform.png?raw=true "Infrastructure as Code - Terraform")

# Set up Terraform
- set up environment variables on local system
  - go to `Environment Variables` click `add`
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`

- Main commands:
  - `init` - Prepare your working directory for other commands
  - `validate` - Check whether the configuration is valid
  - `plan` - Show changes required by the current configuration
  - `apply` - Create or update infrastructure
  - `destroy` - Destroy previously-created infrastructure

- create file `variable.tf` for sensitive info with formatting as follows:

```
variable "aws_key_name" {
    default = "~/.ssh/eng119.pem"
}
```

- create file `main.tf` with formatting as follows:
  
```
# create a script to initialise terraform to download dependencies required for aws

# first step to create block of code to communicate with aws

provider "aws" {
 region = var.aws_region
}

# create a script to launch a VPC - then relaunch this ec2 in your vpc
resource "aws_vpc" "terraform_vpc" {
  cidr_block = var.vpc_cidr
  instance_tenancy = "default"
  enable_dns_hostnames = true

  tags = {
      Name = var.terraform_vpc
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.terraform_vpc.id
  tags = {
    Name = var.terraform_igw
  }
}

resource "aws_subnet" "terraform_subnet" {
  vpc_id = aws_vpc.terraform_vpc.id
  cidr_block = var.vpc_cidr
  depends_on = [aws_internet_gateway.gw]

  tags = {
    Name = var.terraform_subnet
  }
}

resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.terraform_vpc.id

  route {
    cidr_block = var.route_first
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = var.terraform_route_table
  }
}

resource "aws_route_table_association" "rta" {
  subnet_id = aws_subnet.terraform_subnet.id
  route_table_id = aws_route_table.rt.id
}

resource "aws_security_group" "terraform_sg" {
  vpc_id = aws_vpc.terraform_vpc.id

  ingress = {
    cidr_blocks = var.sg_ip_permission
    from_port = "22"
    protocol = "tcp"
    to_port = "22"
  } 

  tags = {
    Name = var.terraform_sg
  }
  
}

resource "aws_instance" "app_instance" {  

  ami = var.node_ami_id

  # what type of instance to launch
  instance_type = var.terraform_instance_type

  # do we want to have it globally available - public ip
  associate_public_ip_address = true

  # name your instance
  tags = {
    Name = var.terraform_instance_name
  }

  # attach the file.pem
  key_name = var.aws_key_name

  # vpc
  subnet_id = aws_subnet.terraform_subnet

  # security group
  vpc_security_group_ids = aws_security_group.terraform_sg

  # Name your instance
  tags = {
    Name = "eng110_benswen_terraform_app"
  }


}

```

