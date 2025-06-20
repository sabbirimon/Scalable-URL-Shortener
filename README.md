# Build a Scalable URL Shortener System with Citus and Redis Cluster (Powered by Poridhi.io Labs)


URL Shortener is a web application that allows users to create short URLs for long URLs. It is a simple and efficient way to shorten long URLs and share them with others. In this lab, we will build a scalable URL shortener system using Citus and Redis Cluster.

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/systemDesign.drawio%20(1).svg)

Before we start, lets understand the components of the system.

## Citus for Horizontal Scaling

Citus is an open-source extension to PostgreSQL that transforms it into a distributed database by sharding data and queries across multiple nodes. It supports horizontal scaling, parallelism, and high availability, making it suitable for your URL shortener’s high-throughput and large-scale data needs. Citus operates with a coordinator node (managing metadata and query routing) and worker nodes (storing data shards).

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/citus.drawio.svg)

### Scaling Strategy in Citus

Citus supports two sharding methods: **row-based sharding** (distributing rows across nodes based on a distribution column) and **schema-based sharding** (distributing entire schemas). For your URL shortener, we’ll use row-based sharding.

- Our **urls** table has a natural distribution column `(short_url_id)`.
- It supports efficient queries for redirects `(GET /:shortUrlId)` and analytics.
- It’s better suited for large, uniform datasets like URL mappings compared to schema-based sharding, which fits multi-tenant apps with isolated schemas.


## PgBouncer for Connection Pooling

PgBouncer is a connection pooler for PostgreSQL. It helps manage database connections efficiently by keeping an array of open connections and reusing them, instead of creating and destroying them for each request. Additionally, PgBouncer has a very low memory footprint—some users report as low as 2 KB per connection, making it an ideal solution for high-throughput applications.

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/db-conn3.drawio.svg)

PgBouncer acts as a proxy between applications and the PostgreSQL database.

## Redis Cluster

Redis Cluster is a distributed system designed for horizontal scalability and high availability by automatically partitioning data across multiple nodes using a hash slot mechanism. It divides the keyspace into 16,384 hash slots, with each master node responsible for a subset of those slots, allowing efficient data sharding. 

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/redis-arch.drawio.svg)

To ensure fault tolerance, each master can have one or more replica nodes that automatically take over in case the master fails, enabling seamless failover. The cluster operates without a central coordinator, relying instead on a gossip protocol for inter-node communication and state sharing, eliminating single points of failure.

## AWS Architecture

This setup includes everything from VPCs and subnets to app servers, Redis Cluster, PostgreSQL with Citus for horizontal scaling, a load balancer, and even a bastion host for secure SSH access.

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/systemDesign-aws.drawio.svg)


### The Network: Custom VPC with Public & Private Subnets

We start with a **custom VPC** named `url-shortener-vpc` with the CIDR block `10.0.0.0/16`. Within this VPC, we define:

- **2 Public Subnets** (`10.0.1.0/24` and `10.0.2.0/24`) — spread across AZs `ap-southeast-1a` and `1b`.
- **2 Private Subnets** (`10.0.3.0/24` and `10.0.4.0/24`) — for our app, database, and internal services.

The **public subnets** host:

- A NAT Gateway (for private subnet egress)
- A Bastion Host (for SSH access)
- The Application Load Balancer (ALB)

The **private subnets** are reserved for:

- Node.js app servers
- Redis Cluster
- PgBouncer
- CitusDB (PostgreSQL) nodes

We wire this all together with:
- An **Internet Gateway** for public internet access
- A **NAT Gateway** in a public subnet for private subnets
- Custom **route tables** for public and private routing

### Security Groups: Layered Access Control

Security is front and center in this setup. Here are the security groups:

Certainly! Here's the security group rule summary in your requested format:

| **Security Group**  | **Ingress Rules**                                                                                                                                                                                                | **Purpose**                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Bastion Host SG** | - SSH (port 22) from the internet                                                                                                                                                                                | Jump server for accessing private instances                  |
| **ALB SG**          | - HTTP (port 80) from anywhere<br>- HTTPS (port 443) from anywhere                                                                                                                                               | Exposes the app to the internet                              |
| **App Server SG**   | - HTTP (port 3000) from ALB<br>- SSH (port 22) from Bastion                                                                                                                                                      | Accepts traffic from ALB and allows admin access via Bastion |
| **PgBouncer SG**    | - PostgreSQL (port 6432) from App Servers<br>- SSH (port 22) from Bastion                                                                                                                                        | Pools database connections for better performance            |
| **Database SG**     | - Redis (6379) from App Servers<br>- PostgreSQL (5432) from PgBouncer<br>- Citus inter-node (5432)<br>- SSH (22) from Bastion<br>- Redis internal traffic (all ports, self)<br>- Redis cluster bus (16379, self) | Secure and segmented access to Redis and PostgreSQL          |

With this strict segmentation, each layer talks only to the services it needs.

### Core Services: App, Redis, and CitusDB

The infrastructure includes two **Node.js application EC2 instances** running in private subnets, each configured with Docker and exposed on port 3000, fronted by a **public Application Load Balancer (ALB)** that performs health checks at `/health` and routes traffic across availability zones. For data persistence, the setup uses **CitusDB**, a distributed PostgreSQL cluster consisting of one coordinator and two worker nodes, all running in Docker. To optimize database connectivity, a **PgBouncer instance** serves as a connection pooler between the app servers and the database layer. For caching, we will use **Redis** cluster.


Now, lets create the infrastructure for the URL shortener system.


## Environment Setup for Infrastructure Provisioning

### Configure AWS CLI

The AWS CLI is a command-line tool that allows you to interact with AWS services programmatically. It simplifies provisioning resources, such as EC2 instances and load balancers, which are required to host our database cluster and application server. Lets configure the AWS CLI.

```bash
aws configure
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/1c840787f4547fa9e5f358506471e7801c379b22/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2004/images/image-1.png)

- `AWS Access Key ID:` Your access key to authenticate AWS API requests.
- `AWS Secret Access Key:` A secret key associated with your access key.
- `Default region:` The AWS region in which you want to provision your resources (ap-southeast-1).
- `Default output format:` You can choose the output format (JSON, text, table).

> Get your access key and secret access key from `Poridhi's lab` account by generating AWS credentials.


### Provisioning Compute Resources

**1. Create a Directory for Your Infrastructure**

Before starting, it’s best to create a dedicated directory for the infrastructure files:

```bash
mkdir url-shortener-infra
cd url-shortener-infra
```

**2. Install Python venv**

Set up a Python virtual environment (venv) to manage dependencies for Pulumi or other Python-based tools:

```bash
sudo apt update
sudo apt install python3.8-venv -y
```

This will set up a Python virtual environment which will be useful later when working with Pulumi.

**3. Initialize a New Pulumi Project**

First login to Pulumi by running the command in the terminal. You will require a **token** to login. You can get the token from the Pulumi website.

```bash
pulumi login
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/1c840787f4547fa9e5f358506471e7801c379b22/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2004/images/image-2.png)

After login, run the following command to initialize a new Pulumi project:

```bash
pulumi new aws-python
```

> You will prompted with serveral questions. Go with the default options. Only change the aws region to `ap-southeast-1`.

**4. Update the `__main.py__` file:**

Open the `__main__.py` file in the project directory and define the AWS infrastructure required for our project. It handles the creation of a Virtual Private Cloud (VPC), subnets, security group, NAT Gateway, Internet Gateway, Route Table, and EC2 instances. We will also inject **user data** to each instances that will install some required tools.


```python
import os
import pulumi
import pulumi_aws as aws

# Configuration
vpc_cidr = "10.0.0.0/16"
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]  # ap-southeast-1a, ap-southeast-1b
private_subnet_cidrs = ["10.0.3.0/24", "10.0.4.0/24"]  # ap-southeast-1a, ap-southeast-1b

instance_type = 't2.micro'
ami = 'ami-01811d4912b4ccb26'  # Ubuntu 22.04 LTS in ap-southeast-1
key_name = "url-shortener"

# Create a VPC
vpc = aws.ec2.Vpc("url-shortener-vpc",
    cidr_block=vpc_cidr,
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={
        "Name": "url-shortener-vpc",
    })

# Create Internet Gateway
igw = aws.ec2.InternetGateway("url-shortener-igw",
    vpc_id=vpc.id,
    tags={
        "Name": "url-shortener-igw",
    })

# Create Public Subnets
public_subnets = []
for i, cidr in enumerate(public_subnet_cidrs):
    az = "ap-southeast-1a" if i == 0 else "ap-southeast-1b"
    subnet = aws.ec2.Subnet(f"public-subnet-{i}",
        vpc_id=vpc.id,
        cidr_block=cidr,
        availability_zone=az,
        map_public_ip_on_launch=True,
        tags={
            "Name": f"public-subnet-{i}",
        })
    public_subnets.append(subnet)

# Create Private Subnets
private_subnets = []
for i, cidr in enumerate(private_subnet_cidrs):
    az = "ap-southeast-1a" if i == 0 else "ap-southeast-1b"
    subnet = aws.ec2.Subnet(f"private-subnet-{i}",
        vpc_id=vpc.id,
        cidr_block=cidr,
        availability_zone=az,
        tags={
            "Name": f"private-subnet-{i}",
        })
    private_subnets.append(subnet)

# Create NAT Gateway in public subnet (for private subnet internet access)
eip = aws.ec2.Eip("nat-eip",
    vpc=True,
    tags={
        "Name": "url-shortener-nat-eip",
    }
)

nat_gateway = aws.ec2.NatGateway("nat-gateway",
    allocation_id=eip.id,
    subnet_id=public_subnets[0].id,
    tags={
        "Name": "url-shortener-nat-gw",
    },
    opts=pulumi.ResourceOptions(depends_on=[igw])
)

# Create Route Tables
public_route_table = aws.ec2.RouteTable("public-route-table",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=igw.id,
        ),
    ],
    tags={
        "Name": "public-route-table",
    })

private_route_table = aws.ec2.RouteTable("private-route-table",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            nat_gateway_id=nat_gateway.id,
        ),
    ],
    tags={
        "Name": "private-route-table",
    })

# Associate Route Tables with Subnets
for i, subnet in enumerate(public_subnets):
    aws.ec2.RouteTableAssociation(f"public-rt-assoc-{i}",
        route_table_id=public_route_table.id,
        subnet_id=subnet.id)

for i, subnet in enumerate(private_subnets):
    aws.ec2.RouteTableAssociation(f"private-rt-assoc-{i}",
        route_table_id=private_route_table.id,
        subnet_id=subnet.id)
    

# Create Bastion Host
bastion_sg = aws.ec2.SecurityGroup("bastion-sg",
    description="Allow SSH from my IP",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            description="SSH from my IP",
            from_port=22,
            to_port=22,
            protocol="tcp",
            cidr_blocks=['0.0.0.0/0'],  # You may set it to your IP address
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            from_port=0,
            to_port=0,
            protocol="-1",
            cidr_blocks=["0.0.0.0/0"],
        ),
    ],
    tags={
        "Name": "url-shortener-bastion-sg",
    })

# Create Security Groups
alb_sg = aws.ec2.SecurityGroup("alb-sg",
    description="Allow HTTP/HTTPS traffic to ALB",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            description="HTTP",
            from_port=80,
            to_port=80,
            protocol="tcp",
            cidr_blocks=["0.0.0.0/0"],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="HTTPS",
            from_port=443,
            to_port=443,
            protocol="tcp",
            cidr_blocks=["0.0.0.0/0"],
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            from_port=0,
            to_port=0,
            protocol="-1",
            cidr_blocks=["0.0.0.0/0"],
        ),
    ],
    tags={
        "Name": "url-shortener-alb-sg",
    })


app_sg = aws.ec2.SecurityGroup("app-sg",
    description="Allow traffic from ALB and SSH from bastion",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            description="HTTP from ALB",
            from_port=3000,
            to_port=3000,
            protocol="tcp",
            security_groups=[alb_sg.id],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="SSH from bastion",
            from_port=22,
            to_port=22,
            protocol="tcp",
            security_groups=[bastion_sg.id],
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            from_port=0,
            to_port=0,
            protocol="-1",
            cidr_blocks=["0.0.0.0/0"],
        ),
    ],
    tags={
        "Name": "url-shortener-app-sg",
    }
)

# Create Security Groups
pgbouncer_sg = aws.ec2.SecurityGroup("pgbouncer-sg",
    description="Allow PostgreSQL traffic from app servers",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            description="PgBouncer from app servers",
            from_port=6432,
            to_port=6432,
            protocol="tcp",
            security_groups=[app_sg.id],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="SSH from bastion",
            from_port=22,
            to_port=22,
            protocol="tcp",
            security_groups=[bastion_sg.id],
        ),
    ],
    egress=[aws.ec2.SecurityGroupEgressArgs(
        from_port=0,
        to_port=0,
        protocol="-1",
        cidr_blocks=["0.0.0.0/0"],
    )],
    tags={"Name": "url-shortener-pgbouncer-sg"}
)

# Update db_sg to only allow PgBouncer to connect to Citus
db_sg = aws.ec2.SecurityGroup("db-sg",
    description="Allow traffic from PgBouncer and Redis clients",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            description="Internal Redis cluster communication",
            from_port=0,  # All ports
            to_port=0,
            protocol="-1",  # All protocols
            self=True,  # Allow instances with this SG to talk to each other
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="Redis from app servers",
            from_port=6379,
            to_port=6379,
            protocol="tcp",
            security_groups=[app_sg.id],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="Redis cluster bus",
            from_port=16379,
            to_port=16379,
            protocol="tcp",
            self=True,  # Allow Redis cluster nodes to communicate
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="PostgreSQL from PgBouncer",
            from_port=5432,
            to_port=5432,
            protocol="tcp",
            security_groups=[pgbouncer_sg.id],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="SSH from bastion",
            from_port=22,
            to_port=22,
            protocol="tcp",
            security_groups=[bastion_sg.id],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            description="Citus worker communication",
            from_port=5432,
            to_port=5432,
            protocol="tcp",
            self=True,
        ),
    ],
    egress=[aws.ec2.SecurityGroupEgressArgs(
        from_port=0,
        to_port=0,
        protocol="-1",
        cidr_blocks=["0.0.0.0/0"],
    )],
    tags={"Name": "url-shortener-db-sg"}
)

bastion = aws.ec2.Instance("bastion-host",
    ami=ami,
    instance_type=instance_type,
    subnet_id=public_subnets[1].id,
    vpc_security_group_ids=[bastion_sg.id],
    associate_public_ip_address=True,
    key_name=key_name,
    tags={
        "Name": "url-shortener-bastion",
})

# Create ALB
alb = aws.lb.LoadBalancer("url-shortener-alb",
    internal=False,
    load_balancer_type="application",
    security_groups=[alb_sg.id],
    subnets=[subnet.id for subnet in public_subnets],
    tags={
        "Name": "url-shortener-alb",
    })

target_group = aws.lb.TargetGroup("app-target-group",
    port=3000,
    protocol="HTTP",
    vpc_id=vpc.id,
    target_type="instance",
    health_check=aws.lb.TargetGroupHealthCheckArgs(
        path="/health",
        port="3000",
        protocol="HTTP",
        healthy_threshold=2,
        unhealthy_threshold=2,
        timeout=3,
        interval=30,
    ),
    tags={
        "Name": "url-shortener-app-tg",
    })

listener = aws.lb.Listener("alb-listener",
    load_balancer_arn=alb.arn,
    port=80,
    protocol="HTTP",
    default_actions=[aws.lb.ListenerDefaultActionArgs(
        type="forward",
        target_group_arn=target_group.arn,
    )])

def generate_app_user_data(hostname: str) -> str:
    return f"""#!/bin/bash
sudo hostnamectl set-hostname {hostname}
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
"""

# Create Node.js App Instances
app_instances = []
for i in range(2):
    instance = aws.ec2.Instance(f"app-instance-{i}",
        ami=ami,
        instance_type=instance_type,
        subnet_id=private_subnets[i].id,
        vpc_security_group_ids=[app_sg.id],
        key_name=key_name,
        user_data=generate_app_user_data(f"app-server-{i+1}"),
        tags={
            "Name": f"url-shortener-app-{i}",
        },
        opts=pulumi.ResourceOptions(
            depends_on=[
                nat_gateway,
            ]
        )
    )
    app_instances.append(instance)

# Register app instances with target group
for i, instance in enumerate(app_instances):
    aws.lb.TargetGroupAttachment(f"tg-attachment-{i}",
        target_group_arn=target_group.arn,
        target_id=instance.id)

redis_cluster_user_data = """#!/bin/bash
# Install Docker
sudo apt-get update -y
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

# Create config with dynamic IP detection
sudo mkdir -p /etc/redis
cat <<EOF | sudo tee /etc/redis/redis.conf
# Cluster
cluster-enabled yes
cluster-config-file /data/nodes.conf
cluster-node-timeout 5000
cluster-announce-ip $(hostname -I | awk '{print $1}')
cluster-announce-port 6379
cluster-announce-bus-port 16379

# Security
requirepass your_redis_password
masterauth your_redis_password
rename-command FLUSHDB ""
rename-command FLUSHALL ""

# Persistence
appendonly yes
appendfsync everysec

# Networking
bind 0.0.0.0
protected-mode no

# Performance
maxmemory 1gb
maxmemory-policy allkeys-lru
EOF

# Run Redis with config and persistent storage
sudo docker run -d --name redis \\
  -p 6379:6379 \\
  -p 16379:16379 \\
  -v /etc/redis/redis.conf:/usr/local/etc/redis/redis.conf \\
  -v /data:/data \\
  redis:latest \\
  redis-server /usr/local/etc/redis/redis.conf
"""

# Create Redis Cluster (3 master nodes and 3 replica nodes)
redis_master_instances = []
redis_replica_instances = []
redis_subnet_ids = [private_subnets[0].id, private_subnets[1].id] 

# Master nodes
for i in range(3):
    instance = aws.ec2.Instance(f"redis-master-{i+1}",
        ami=ami,
        instance_type=instance_type,
        subnet_id=redis_subnet_ids[i % len(redis_subnet_ids)],  # Distribute across AZs
        vpc_security_group_ids=[db_sg.id],
        key_name=key_name,
        user_data=redis_cluster_user_data,
        tags={
            "Name": f"url-shortener-redis-master-{i+1}",
        },
        opts=pulumi.ResourceOptions(
            depends_on=[nat_gateway]
        )
    )
    redis_master_instances.append(instance)

redis_master_private_ips = [instance.private_ip for instance in redis_master_instances]

# Replica nodes
for i in range(3):
    instance = aws.ec2.Instance(f"redis-replica-{i+1}",
        ami=ami,
        instance_type=instance_type,
        subnet_id=redis_subnet_ids[i % len(redis_subnet_ids)],  # Distribute across AZs
        vpc_security_group_ids=[db_sg.id],
        key_name=key_name,
        user_data=redis_cluster_user_data,
        tags={
            "Name": f"url-shortener-redis-replica-{i+1}",
        },
        opts=pulumi.ResourceOptions(
            depends_on=[nat_gateway]
        )
    )
    redis_replica_instances.append(instance)

# Output the Redis cluster IPs for configuration
redis_master_private_ips = [instance.private_ip for instance in redis_master_instances]
redis_replica_private_ips = [instance.private_ip for instance in redis_replica_instances]

# PgBouncer EC2 Instance
pgbouncer_instance = aws.ec2.Instance("pgbouncer",
    ami=ami,  
    instance_type=instance_type,
    subnet_id=private_subnets[0].id, #ap-southeast-1a
    vpc_security_group_ids=[pgbouncer_sg.id],
    key_name=key_name,
    user_data="""#!/bin/bash
        sudo hostnamectl set-hostname pgbouncer
        apt update && apt install -y pgbouncer postgresql-client
        systemctl enable pgbouncer
        systemctl start pgbouncer
    """,
    tags={"Name": "url-shortener-pgbouncer"},
    opts=pulumi.ResourceOptions(
        depends_on=[
            nat_gateway,
        ]
    )
)

def generate_citus_user_data(hostname: str) -> str:
    return f"""#!/bin/bash
sudo hostnamectl set-hostname {hostname}
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
"""

# Create Citus Coordinator
coordinator_instance = aws.ec2.Instance("citus-coordinator",
    ami=ami,
    instance_type=instance_type,
    subnet_id=private_subnets[0].id, #ap-southeast-1a
    vpc_security_group_ids=[db_sg.id],
    key_name=key_name,
    user_data=generate_citus_user_data("citus-coordinator"),
    tags={
        "Name": "url-shortener-citus-coordinator",
    },
    opts=pulumi.ResourceOptions(
        depends_on=[
            nat_gateway,
        ]
    )
)

# Create Citus Workers
worker_instances = []
for i in range(2):
    instance = aws.ec2.Instance(f"citus-worker-{i+1}",
        ami=ami,
        instance_type=instance_type,
        subnet_id=private_subnets[i].id,
        vpc_security_group_ids=[db_sg.id],
        key_name=key_name,
        user_data=generate_citus_user_data(f"citus-worker-{i+1}"),
        tags={
            "Name": f"url-shortener-citus-worker-{i}",
        },
        opts=pulumi.ResourceOptions(
            depends_on=[
                nat_gateway,
            ]
        )
    )
    worker_instances.append(instance)

# Output important information
pulumi.export("vpc_id", vpc.id)
pulumi.export("alb_dns_name", alb.dns_name)
pulumi.export("bastion_public_ip", bastion.public_ip)
pulumi.export("app_instance_ids", [instance.id for instance in app_instances])
pulumi.export("citus_coordinator_id", coordinator_instance.id)
pulumi.export("citus_worker_ids", [instance.id for instance in worker_instances])
pulumi.export("redis_master_instance_ids", [instance.id for instance in redis_master_instances])
pulumi.export("redis_replica_instance_ids", [instance.id for instance in redis_replica_instances])

pulumi.export("app_private_ips", [instance.private_ip for instance in app_instances])
pulumi.export("redis_master_private_ips", redis_master_private_ips)
pulumi.export("redis_replica_private_ips", redis_replica_private_ips)
pulumi.export("pgbouncer_private_ip", pgbouncer_instance.private_ip)
pulumi.export("citus_coordinator_private_ip", coordinator_instance.private_ip)
pulumi.export("citus_worker_private_ips", [instance.private_ip for instance in worker_instances])

# Create config file for SSH access
def create_config_file(ips):
    bastion_host = 'bastion-server'

    # Map hostnames to the correct IPs
    private_host_map = {
        'app-server-1': ips['app_private_ips'][0],
        'app-server-2': ips['app_private_ips'][1],
        'citus-coordinator-server': ips['citus_coordinator_private_ip'],
        'citus-worker-server-1': ips['citus_worker_private_ips'][0],
        'citus-worker-server-2': ips['citus_worker_private_ips'][1],
        'pgbouncer-server': ips['pgbouncer_private_ip'],
        'redis-master-1': ips['redis_master_private_ips'][0],
        'redis-master-2': ips['redis_master_private_ips'][1],
        'redis-master-3': ips['redis_master_private_ips'][2],
        'redis-replica-1': ips['redis_replica_private_ips'][0],
        'redis-replica-2': ips['redis_replica_private_ips'][1],
        'redis-replica-3': ips['redis_replica_private_ips'][2],
    }

    # Start building SSH config
    config_content = f"""Host {bastion_host}
    HostName {ips['bastion_public_ip']}
    User ubuntu
    IdentityFile ~/.ssh/{key_name}.id_rsa

"""

    for hostname, ip in private_host_map.items():
        config_content += f"""Host {hostname}
    ProxyJump {bastion_host}
    HostName {ip}
    User ubuntu
    IdentityFile ~/.ssh/{key_name}.id_rsa

"""

    config_path = os.path.expanduser("~/.ssh/config") # Static Path. You need to change it according to your system.

    # Ensure the .ssh directory exists
    os.makedirs(os.path.dirname(config_path), exist_ok=True)

    # Write the SSH config file
    with open(config_path, "w") as config_file:
        config_file.write(config_content)

    # Optional: Secure the file (Linux/Mac)
    os.chmod(config_path, 0o600)


# Collect the IPs for all nodes
all_ips = pulumi.Output.all(
    bastion_public_ip=bastion.public_ip,
    app_private_ips=pulumi.Output.all(*[instance.private_ip for instance in app_instances]),
    redis_master_private_ips=pulumi.Output.all(*[instance.private_ip for instance in redis_master_instances]),
    redis_replica_private_ips=pulumi.Output.all(*[instance.private_ip for instance in redis_replica_instances]),
    citus_coordinator_private_ip=coordinator_instance.private_ip,
    citus_worker_private_ips=pulumi.Output.all(*[instance.private_ip for instance in worker_instances]),
    pgbouncer_private_ip=pgbouncer_instance.private_ip
)

# Apply the config creation when outputs are resolved
all_ips.apply(lambda ips: create_config_file(ips))
```

**5. Create an AWS Key Pair**

This key pair will be used to authenticate when accessing EC2 instances. Use the following AWS CLI command to create a new SSH key pair named `url-shortener`:


```bash
cd ~/.ssh/
aws ec2 create-key-pair --key-name url-shortener --output text --query 'KeyMaterial' > url-shortener.id_rsa
chmod 400 url-shortener.id_rsa
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-1.png)

This will save the private key as `url-shortener.id_rsa` in the `~/.ssh/` directory and restrict its permissions.


**6. Provision the Infrastructure**

Run the following command to provision the infrastructure:

```bash
pulumi up --yes
```

![alt text](./images/image.png)

Here we can see the infrastructure is provisioned successfully. Now we can use the config file to access the instances. Check the file config file content to get the hostnames of the instances.

```bash
cat ~/.ssh/config
```

![alt text](./images/image-5.png)

**7. SSH into the EC2 instance**

We can use the config file to access the app server.

```bash
ssh app-server-1
```

You may change the hostname to `app-server-1` to make it easier to identify the server. This is optional.

```bash
sudo hostnamectl set-hostname app-server-1
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-4.png)

In the same way, we can ssh into the other EC2 instances.


## Citus Cluster Setup

First SSH into the **coordinator** instance. Then check if docker is running. As we have already installed docker in the user data, it will start automatically.

```bash
sudo docker ps
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-6.png)

If you don't see any output, it means docker is not running or not installed yet. Wait for a few minutes and check again. You can check the cloud-init logs to see if docker is installed.

```bash
sudo tail -f /var/log/cloud-init-output.log
```

### Coordinator Node Setup

To setup the coordinator node, we need to create a **docker-compose** file and a `.env` file.

**1. Create a docker-compose file for the coordinator node.**

```bash
version: '3.8'

services:
  citus-coordinator:
    image: citusdata/citus:11.3.0
    environment:
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - POSTGRES_DB=${PG_DATABASE}
    volumes:
      - citus-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PG_USER} -d ${PG_DATABASE}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  citus-data:
```

**2. Create .env file:**

```bash
PG_USER=your_pg_user
PG_PASSWORD=your_pg_password
PG_DATABASE=url_shortener
```

> You may change the `PG_USER` and `PG_PASSWORD` to your own. But make sure to update the same in the `.env` file of the worker node and all other places where we are using the same credentials.

Start the service:

```bash
docker-compose up -d
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-7.png)

### Worker Node Setup

**1. Create a docker-compose file for the worker node.**

```bash
version: '3.8'

services:
  citus-worker:
    image: citusdata/citus:11.3.0
    environment:
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - POSTGRES_DB=${PG_DATABASE}
    volumes:
      - citus-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PG_USER} -d ${PG_DATABASE}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  citus-data:
```

Create identical `.env` file and start:

```bash
docker-compose up -d
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-8.png)

### Initialize Citus Cluster

Run this command in the **coordinator** instance:

```bash
docker-compose exec citus-coordinator psql -U your_pg_user -d url_shortener
```

In the PostgreSQL shell, run the following commands to enable the Citus extension and add the worker nodes.

```sql
-- Enable Citus extension
CREATE EXTENSION citus;

-- Add worker nodes (use private IPs)
SELECT citus_add_node('<worker1-private-ip>', 5432);
SELECT citus_add_node('<worker2-private-ip>', 5432);

-- Verify setup
SELECT * FROM citus_get_active_worker_nodes();
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-9.png)


### Create urls table

In the same psql shell, run the following commands to create the urls table.

```sql
-- Create urls table
CREATE TABLE urls (
    short_url_id VARCHAR(7) PRIMARY KEY,
    long_url TEXT NOT NULL,
    user_id VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Distribute urls table
SELECT create_distributed_table('urls', 'short_url_id', colocate_with => 'none');

-- Verify distribution
SELECT * FROM citus_tables;
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-10.png)

> Make sure to update the worker's private IP address.

### Create keys table

In the same psql shell, run the following commands to create the keys table.

**1. First, create the keys table if it doesn't exist:**

```sql
CREATE TABLE IF NOT EXISTS keys (
  short_url_id VARCHAR(7) PRIMARY KEY,
  used BOOLEAN DEFAULT FALSE
);
```

**2. SQL Function to generate random keys**

```sql
CREATE OR REPLACE FUNCTION generate_random_key(length INTEGER DEFAULT 7) 
RETURNS TEXT AS $$
DECLARE
  chars TEXT := 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  result TEXT := '';
  i INTEGER;
BEGIN
  FOR i IN 1..length LOOP
    result := result || substr(chars, floor(random() * length(chars) + 1)::INTEGER, 1);
  END LOOP;
  RETURN result;
END;
$$ LANGUAGE plpgsql;
```

**3. SQL Procedure to populate keys**


```sql
CREATE OR REPLACE PROCEDURE populate_keys(total_count INTEGER, batch_size INTEGER DEFAULT 10000)
AS $$
DECLARE
  i INTEGER;
  generated_key TEXT;
BEGIN
  FOR i IN 1..(total_count/batch_size) LOOP
    BEGIN
      FOR j IN 1..batch_size LOOP
        generated_key := generate_random_key();
        INSERT INTO keys (short_url_id, used) 
        VALUES (generated_key, FALSE)
        ON CONFLICT (short_url_id) DO NOTHING;
      END LOOP;
      
      RAISE NOTICE 'Inserted % keys...', i * batch_size;
    EXCEPTION
      WHEN OTHERS THEN
        RAISE WARNING 'Error in batch %: %', i, SQLERRM;
    END;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

**4. Execute the procedure to generate 1,000,000 keys:**

```sql
CALL populate_keys(1000000);
```

You can check the keys table to see the keys are generated successfully.

```sql
SELECT * FROM keys limit 10;
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-11.png)


## Configure PgBouncer

SSH into the **PgBouncer** instance and configure it to connect to the Citus cluster. We have already installed pgbouncer by injecting user data to the EC2 instance. Now we have to configure it to connect to the Citus cluster.


**1. Configure PgBouncer**

Edit the main configuration file:

```bash
sudo vim /etc/pgbouncer/pgbouncer.ini
```

Use this configuration (adjust for your Citus setup):

```ini
[databases]
url_shortener = host=<COORDINATOR-PRIVATE-IP> port=5432 dbname=url_shortener

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
ignore_startup_parameters = extra_float_digits
server_reset_query = DISCARD ALL
log_connections = 1
log_disconnections = 1
```

**2. Create Authentication File**

```bash
sudo vim /etc/pgbouncer/userlist.txt
```

Add your credentials (format: `"username" "password"`):

```text
"your_pg_user" "your_pg_password"
```

> You may change the username and password to your own and also convert the password to md5 hash. But for now we want to keep it simple.

Set proper permissions:

```bash
sudo chown postgres:postgres /etc/pgbouncer/userlist.txt
sudo chmod 640 /etc/pgbouncer/userlist.txt
```

**3. Enable and Start the Service**

```bash
sudo systemctl daemon-reload
sudo systemctl enable pgbouncer
sudo systemctl restart pgbouncer
```

**4. Verify the Service**

Check status:

```bash
sudo systemctl status pgbouncer
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-12.png)

Test connection:
```bash
psql -h localhost -p 6432 -U your_pg_user -d url_shortener -c "SELECT 1"
```

> Provide the credentials and check if the connection is successful.

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-13.png)


> **Troubleshooting**

Make sure the PgBouncer service is running:

```bash
sudo systemctl status pgbouncer
```

Using netcat to check if port is open

```bash
# Using netcat to check if port is open
nc -zv localhost 6432

# Or using telnet
sudo apt install -y telnet
telnet localhost 6432
# (Then type Q to quit)
```

**Test connection to the coordinator:**

```bash
psql -h <COORDINATOR-PRIVATE-IP> -p 5432 -U your_pg_user -d url_shortener -c "SELECT 1"
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-14.png)


## Redis Cluster Setup

Now we need to setup the Redis cluster. We have 6 instances of Redis. 3 masters and 3 replicas. We will use the private IPs of the instances to setup the cluster. We have already run the redis-server on all the instances as a docker container. First SSH into one of the Redis master nodes.

```bash
ssh redis-master-1
```

Check the status of the docker container.

```bash
docker ps
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/image-1.png)


if everything is running fine, we can proceed to setup the cluster.

### **1. Install `redis-tools` (if not already available)**
You need `redis-cli` to initialize the cluster.

```bash
sudo apt update
sudo apt install -y redis-tools
```

### **2. Prepare the Cluster Creation Command**
You need the private IPs of all 6 nodes (3 masters + 3 replicas).  
Run this command **from any one of the master nodes**:

```bash
REDISCLI_AUTH='your_redis_password' redis-cli --cluster create \
  <MASTER_1_IP>:6379 \
  <MASTER_2_IP>:6379 \
  <MASTER_3_IP>:6379 \
  <REPLICA_1_IP>:6379 \
  <REPLICA_2_IP>:6379 \
  <REPLICA_3_IP>:6379 \
  --cluster-replicas 1
```

> Replace the password with your own password. Or to keep things simple, you may use the password `your_redis_password` to make all configurations easier.

**What `--cluster-replicas 1` does:**  
- Assigns 1 replica per master automatically.
- The first 3 IPs become masters, the next 3 become replicas.


![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/image-6.png)

### **3. Confirm the Cluster Setup**

When you run the command, Redis will propose a configuration.  
Type `yes` to accept it.

### **4. Verify Cluster Health**

Run this from any Redis node:

```bash
REDISCLI_AUTH='your_redis_password' redis-cli --cluster check <Any_Redis_Instance_IP>:6379
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/image-3.png)

> Replace the password with your own password. Or to keep things simple, you may use the password `your_redis_password` to make all configurations easier.


## App Server Setup


SSH into the **app-server-1** instance. We will run our Node.js application using docker. We have already installed docker by injecting user data to the EC2 instance. You may use your own application image or use this application to test the setup. We have already shown how we can implement the application in the previous lab.

Create a .env file and update the environment variables.


```bash
PORT=3000
PG_HOST=<PgBouncer_Private_IP>
PG_PORT=6432
PG_USER=your_pg_user
PG_PASSWORD=your_pg_password
PG_DATABASE=url_shortener
REDIS_NODE1=<Redis_Master_1_Private_IP>
REDIS_NODE2=<Redis_Master_2_Private_IP>
REDIS_NODE3=<Redis_Master_3_Private_IP>
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
BASE_URL=http://<alb-dns-name>/api/urls
```

Run the application:

```bash
sudo docker run --env-file .env -p 3000:3000 konami98/url-shortener-backend:v5
```

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/refs/heads/main/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2007/images/image-4.png)

**Do the same for the other app server.**


## Test the setup

To test URL shortener APIs (`POST /api/urls` and `GET /api/urls/:shortUrlId`) using **Postman**, follow these steps.

### Step 1: Test the `POST /api/urls` Endpoint

This endpoint creates a short URL from a long URL.

#### Setup in Postman

1. **Open Postman**:
   Launch Postman and create a new request.

2. **Set Request Type and URL**:
   - **Method**: Select `POST` from the dropdown.
   - **URL**: Enter `http://<alb-dns-name>/api/urls`.

3. **Set Body**:
   - Go to the **Body** tab.
   - Select `raw` and choose `JSON` from the dropdown.
   - Enter the following JSON payload:
     ```json
     {
       "longUrl": "https://www.example.com/very/long/url/to/test"
     }
     ```

4. **Send the Request**:
   - Click the **Send** button.

#### Expected Response

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-16.png)

**Success (201 Created)**:
- Status: `201 Created`
- Note the `shortUrl` value (e.g., `abc1234`) for the next test.

### Step 2: Test the `GET /api/urls/:shortUrlId` Endpoint

This endpoint redirects from a short URL ID to the long URL.

#### Setup in Postman

1. **Create a New Request**:
   Click the **+** button in Postman to create a new tab.

2. **Set Request Type and URL**:
   - **Method**: Select `GET` from the dropdown.
   - **URL**: Enter `http://<alb-dns-name>/api/urls/<shortUrlId>`, replacing `<shortUrlId>` with the ID from the `POST` response (e.g., `http://<alb-dns-name>/api/urls/abc1234`).

3. **Send the Request**:
   - Click the **Send** button.

#### Expected Response

![alt text](https://raw.githubusercontent.com/poridhiEng/lab-asset/b49eee7ba3facf9079707081bdcaea2ba7a0d4bf/System%20Design%20Labs%20Using%20AWS/Lab%2008/URL_shortener/Lab%2005/images/image-17.png)

**Success (200 OK)**:
- Status: `200 OK`
- Redirects to the long URL (e.g., `https://www.example.com/very/long/url/to/test`).


## Cleanup

To cleanup the infrastructure, run the following command:

```bash
pulumi destroy --yes
```

## Conclusion

We have successfully setup the URL shortener system using AWS services. We have also explored the concepts of Citus, PgBouncer, and Redis Cluster. We have also learned how to load test the setup using k6. In the next lab, we will explore how we can monitor the setup using OpenTelemetry and Grafana.



