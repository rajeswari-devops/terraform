provider "aws" {
  region = "us-east-1"
}
resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "main"
  }
}
resource "aws_subnet" "subnet-A" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "subnet-A"
  }
}

resource "aws_subnet" "subnet-B" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "subnet-B"
  }
}
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "public-route-table"
  }
}
resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.gw.id
}

resource "aws_route_table_association" "subnet_A" {
  subnet_id      = aws_subnet.subnet-A.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "subnet_B" {
  subnet_id      = aws_subnet.subnet-B.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP traffic to ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "alb-sg"
  }
}


resource "aws_security_group" "ec2_sg" {
  name        = "ec2-sg"
  description = "Allow HTTP only from ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB security group"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  egress {
    description = "Allow outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ec2-sg"
  }
}

resource "aws_instance" "server1" {
  ami                         = "ami-04b938437b69a2c9c"
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.subnet-A.id
  associate_public_ip_address = true

  vpc_security_group_ids = [
    aws_security_group.ec2_sg.id
  ]

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y nginx || yum install -y nginx
    echo "<h1>Hello from Server 1</h1>" > /usr/share/nginx/html/index.html
    systemctl enable --now nginx
  EOF

  tags = {
    Name = "server1"
  }
}

resource "aws_instance" "server2" {
  ami                         = "ami-04b938437b69a2c9c"
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.subnet-B.id
  associate_public_ip_address = true

  vpc_security_group_ids = [
    aws_security_group.ec2_sg.id
  ]

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y nginx || yum install -y nginx
    echo "<h1>Hello from Server 2</h1>" > /usr/share/nginx/html/index.html
    systemctl enable --now nginx
  EOF

  tags = {
    Name = "server2"
  }
}

resource "aws_lb_target_group" "app_tg" {
  name     = "app-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
  }

  tags = {
    Name = "app-target-group"
  }
}


resource "aws_lb_target_group_attachment" "server1" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.server1.id
  port             = 80
}


resource "aws_lb_target_group_attachment" "server2" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.server2.id
  port             = 80
}


resource "aws_lb" "app" {
  name               = "my-application-alb"
  internal           = false
  load_balancer_type = "application"

  security_groups = [
    aws_security_group.alb_sg.id
  ]

  subnets = [
    aws_subnet.subnet-A.id,
    aws_subnet.subnet-B.id
  ]

  tags = {
    Name = "my-application-alb"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

output "alb_dns_name" {
  description = "Application Load Balancer DNS name"
  value       = "http://${aws_lb.app.dns_name}"
}
