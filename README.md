# Ansible_aws_infrastructure

## Ansible - This is an ansible playbook to setup an infrastructure in aws. This will create a new VPC and 2 public subnets and 1 private subnets.

## Also creates a new internet gateway for the VPC and one NAT gateway with EIP for private subnet. Then it will create 3 servers. One will be the bastion server which will be used to communicate with the other servers.

## Next a webserver and a database server. Databse server will be created under the private subnetso that there will be no connection from outside but the server can communicate to the outer world using NAT.


```ansible
---
- name: AWS Infrastructure
  hosts: localhost

  vars:
    aws_region: us-east-2
    vpc_name: blog
    vpc_cidr_block: 172.16.0.0/16
    public_subnet_1_cidr: 172.16.0.0/20
    public_subnet_2_cidr:  172.16.16.0/20
    private_subnet_1_cidr: 172.16.32.0/20
    instance_type: t2.micro
    image: ami-0d8f6eb4f641ef691
    keypair: ansible

  tasks:

    - name: Creating new VPC
      ec2_vpc_net:
        name:       "{{ vpc_name }}"
        cidr_block: "{{ vpc_cidr_block }}"
        region:     "{{ aws_region }}"
        state:      "present"
      register: my_vpc

    - name: Wait for 5 Seconds
      wait_for: timeout=5

    - name: Set VPC ID in variable
      set_fact:
        vpc_id: "{{ my_vpc.vpc.id }}"

    - name: Creating Public Subnet1
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_1_cidr }}"
        az:     "{{ aws_region }}a"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-1"
      register: public_subnet1

    - name: Set variable
      set_fact:
        public_subnet_1: "{{ public_subnet1.subnet.id }}"

    - name: Creating Public Subnet2
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_2_cidr }}"
        az:     "{{ aws_region }}b"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-2"
      register: public_subnet2

    - name: Set Public Subnet ID in variable [AZ-2]
      set_fact:
        public_subnet_2: "{{ public_subnet2.subnet.id }}"

    - name: Creating Private Subnet [AZ-1]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ private_subnet_1_cidr }}"
        az:     "{{ aws_region }}c"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Private Subnet-1"
      register: private_subnet3

    - name: Set Private Subnet ID in variable [AZ-3]
      set_fact:
        private_subnet_3: "{{ private_subnet3.subnet.id }}"

    - name: Create Internet Gateway for VPC
      ec2_vpc_igw:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        state:  "present"
      register: blog_igw

    - name: Set Internet Gateway ID in variable
      set_fact:
        igw_id: "{{ blog_igw.gateway_id }}"

    - name: Creating a new NAT gateway with EIP
      ec2_vpc_nat_gateway:
        state: present
        subnet_id: "{{ public_subnet_2 }}"
        wait: yes
        region:    "{{ aws_region }}"
        if_exist_do_not_create: true
      register: blog_nat

    - name: Set Nat Gateway ID in variable 
      set_fact:
        blog_nat_gw: "{{ blog_nat.nat_gateway_id }}"

    - name: Wait for 5 Seconds
      wait_for: timeout=5

    - name: Set up public subnet route table
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Public"
        subnets:
          - "{{ public_subnet_1 }}"
          - "{{ public_subnet_2 }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ igw_id }}"

    - name: Set up private subnet route table 
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Private"
        subnets:
          - "{{ private_subnet_3 }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ blog_nat_gw }}"

    - name: Creating Bastion Security Group
      ec2_group:
        name:  "Bastion-sg"
        description: "Bastion-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0
      register: bastion

    - name: Set Bastion Security Group ID in variable
      set_fact:
        bastion_id: "{{ bastion.group_id }}"

    - name: Creating Webserver Security Group
      ec2_group:
        name:  "Webserver"
        description: "Webserver"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: 0.0.0.0/0
          - proto: "tcp"
            from_port: "22"
            to_port: "22"
            group_id: "{{ bastion_id }}"
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0

      register: webserver_sg

    - name: Set Main Security Group ID in variable
      set_fact:
        webserver_sg_id: "{{ webserver_sg.group_id }}"

    - name: Creating Database Server Security Group
      ec2_group:
        name: "DB"
        description: "DataBase-Server"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: "tcp"
            from_port: "22"
            to_port: "22"
            group_id: "{{ bastion_sg }}"
          - proto: "tcp"
            from_port: 3306
            to_port: 3306
            group_id: "{{ webserver_sg_id }}"
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0

    - name: "Launching Webserver Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: Webserver
        vpc_subnet_id: "{{ public_subnet_2 }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: Webserver
        instance_tags:
          Name: Webserver
        exact_count: 1

    - name: "Launching Bastion Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: Bastion-sg
        vpc_subnet_id: "{{ public_subnet_1 }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: Bastion
        instance_tags:
          Name: Bastion
        exact_count: 1

    - name: "Launching Database Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: DB
        vpc_subnet_id: "{{ private_subnet_3 }}"
        wait: yes
        count_tag:
          Name: DB-Server
        instance_tags:
          Name: DB-Server
        exact_count: 1

```
