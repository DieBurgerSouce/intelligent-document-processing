# Infrastructure as Code (IaC) Guide

## 🎯 Überblick

Dieses Dokument beschreibt die Infrastructure as Code (IaC) Strategie für das Ablage-System. Wir nutzen einen Multi-Tool-Ansatz:

- **Terraform** für Cloud-Ressourcen Provisioning
- **Ansible** für Konfigurationsmanagement
- **Kubernetes Manifests** für Container-Orchestrierung
- **Docker Compose** für lokale Entwicklung

---

## 📦 Terraform Setup

### Provider-Konfiguration

```hcl
# terraform/main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }

  backend "s3" {
    bucket         = "ablage-system-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-central-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "Ablage-System"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

### Variablen-Definition

```hcl
# terraform/variables.tf

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "eu-central-1"
}

variable "gpu_instance_type" {
  description = "GPU instance type for OCR backends"
  type        = string
  default     = "g5.2xlarge"  # NVIDIA A10G, 24GB VRAM
}

variable "cpu_instance_type" {
  description = "CPU instance type for fallback backend"
  type        = string
  default     = "c6i.4xlarge"  # 16 vCPUs, 32GB RAM
}

variable "db_instance_class" {
  description = "RDS PostgreSQL instance class"
  type        = string
  default     = "db.r6g.2xlarge"  # 8 vCPUs, 64GB RAM
}

variable "redis_node_type" {
  description = "ElastiCache Redis node type"
  type        = string
  default     = "cache.r6g.large"  # 2 vCPUs, 13.07GB RAM
}
```

---

## 🖥️ Compute Ressourcen

### GPU-Instanzen für DeepSeek & GOT-OCR

```hcl
# terraform/modules/gpu_compute/main.tf

resource "aws_instance" "gpu_backend" {
  count                  = var.gpu_backend_count
  ami                    = data.aws_ami.ubuntu_gpu.id
  instance_type          = var.gpu_instance_type
  subnet_id              = var.subnet_ids[count.index % length(var.subnet_ids)]
  vpc_security_group_ids = [aws_security_group.gpu_backend.id]
  iam_instance_profile   = aws_iam_instance_profile.gpu_backend.name

  root_block_device {
    volume_size = 200  # GB für OS + Models
    volume_type = "gp3"
    encrypted   = true
  }

  # Zusätzliches Volume für Model Cache
  ebs_block_device {
    device_name = "/dev/sdf"
    volume_size = 500  # GB für Hugging Face Model Cache
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = templatefile("${path.module}/user_data_gpu.sh", {
    environment = var.environment
    cuda_version = "12.1"
  })

  tags = {
    Name = "ablage-gpu-backend-${count.index + 1}"
    Type = "GPU-OCR-Backend"
  }
}

# GPU-AMI mit vorinstalliertem CUDA
data "aws_ami" "ubuntu_gpu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Security Group für GPU-Backend
resource "aws_security_group" "gpu_backend" {
  name_prefix = "ablage-gpu-backend-"
  vpc_id      = var.vpc_id

  ingress {
    description = "FastAPI from ALB"
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  ingress {
    description = "SSH from Bastion"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.bastion_cidr]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ablage-gpu-backend-sg"
  }
}
```

### CPU-Instanzen für Surya+Docling

```hcl
# terraform/modules/cpu_compute/main.tf

resource "aws_autoscaling_group" "cpu_backend" {
  name                = "ablage-cpu-backend-${var.environment}"
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = [aws_lb_target_group.cpu_backend.arn]
  health_check_type   = "ELB"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 3

  launch_template {
    id      = aws_launch_template.cpu_backend.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "ablage-cpu-backend"
    propagate_at_launch = true
  }
}

resource "aws_launch_template" "cpu_backend" {
  name_prefix   = "ablage-cpu-backend-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.cpu_instance_type

  vpc_security_group_ids = [aws_security_group.cpu_backend.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.cpu_backend.name
  }

  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size = 100
      volume_type = "gp3"
      encrypted   = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/user_data_cpu.sh", {
    environment = var.environment
  }))
}
```

---

## 🗄️ Database Infrastructure

### RDS PostgreSQL

```hcl
# terraform/modules/database/main.tf

resource "aws_db_instance" "postgres" {
  identifier     = "ablage-db-${var.environment}"
  engine         = "postgres"
  engine_version = "16.1"

  instance_class    = var.db_instance_class
  allocated_storage = 500  # GB
  storage_type      = "gp3"
  storage_encrypted = true

  db_name  = "ablage_system"
  username = "ablage_admin"
  password = random_password.db_password.result

  multi_az               = var.environment == "prod"
  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  vpc_security_group_ids = [aws_security_group.postgres.id]

  backup_retention_period = var.environment == "prod" ? 30 : 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  deletion_protection = var.environment == "prod"
  skip_final_snapshot = var.environment != "prod"
  final_snapshot_identifier = var.environment == "prod" ? "ablage-db-final-${formatdate("YYYY-MM-DD-hhmm", timestamp())}" : null

  tags = {
    Name = "ablage-postgres-${var.environment}"
  }
}

# DB Subnet Group
resource "aws_db_subnet_group" "postgres" {
  name       = "ablage-db-subnet-group-${var.environment}"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "ablage-db-subnet-group"
  }
}

# Security Group
resource "aws_security_group" "postgres" {
  name_prefix = "ablage-postgres-"
  vpc_id      = var.vpc_id

  ingress {
    description     = "PostgreSQL from Application"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  tags = {
    Name = "ablage-postgres-sg"
  }
}

# Secret Manager für DB Credentials
resource "aws_secretsmanager_secret" "db_password" {
  name = "ablage-db-password-${var.environment}"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = aws_db_instance.postgres.username
    password = random_password.db_password.result
    host     = aws_db_instance.postgres.address
    port     = aws_db_instance.postgres.port
    dbname   = aws_db_instance.postgres.db_name
  })
}

resource "random_password" "db_password" {
  length  = 32
  special = true
}
```

### ElastiCache Redis

```hcl
# terraform/modules/cache/main.tf

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "ablage-redis-${var.environment}"
  replication_group_description = "Redis cluster for Ablage System"

  engine               = "redis"
  engine_version       = "7.0"
  node_type            = var.redis_node_type
  num_cache_clusters   = var.environment == "prod" ? 3 : 2
  port                 = 6379

  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_password.result

  automatic_failover_enabled = var.environment == "prod"
  multi_az_enabled           = var.environment == "prod"

  snapshot_retention_limit = var.environment == "prod" ? 5 : 1
  snapshot_window          = "03:00-05:00"
  maintenance_window       = "sun:05:00-sun:06:00"

  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_slow_log.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }

  tags = {
    Name = "ablage-redis-${var.environment}"
  }
}

resource "aws_elasticache_subnet_group" "redis" {
  name       = "ablage-redis-subnet-group-${var.environment}"
  subnet_ids = var.private_subnet_ids
}

resource "aws_security_group" "redis" {
  name_prefix = "ablage-redis-"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Redis from Application"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }
}

resource "random_password" "redis_password" {
  length  = 32
  special = false  # Redis AUTH doesn't support all special chars
}
```

---

## 💾 Storage Infrastructure

### S3 Buckets (MinIO Alternative)

```hcl
# terraform/modules/storage/main.tf

# Original Documents Bucket
resource "aws_s3_bucket" "original_documents" {
  bucket = "ablage-original-docs-${var.environment}"

  tags = {
    Name        = "Original Documents"
    Retention   = "90-days"
  }
}

resource "aws_s3_bucket_versioning" "original_documents" {
  bucket = aws_s3_bucket.original_documents.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "original_documents" {
  bucket = aws_s3_bucket.original_documents.id

  rule {
    id     = "delete-old-documents"
    status = "Enabled"

    expiration {
      days = 90
    }
  }

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "GLACIER"
    }
  }
}

resource "aws_s3_bucket_encryption" "original_documents" {
  bucket = aws_s3_bucket.original_documents.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Processed Documents Bucket
resource "aws_s3_bucket" "processed_documents" {
  bucket = "ablage-processed-docs-${var.environment}"
}

resource "aws_s3_bucket_lifecycle_configuration" "processed_documents" {
  bucket = aws_s3_bucket.processed_documents.id

  rule {
    id     = "delete-old-processed"
    status = "Enabled"

    expiration {
      days = 30
    }
  }
}

# Thumbnails Bucket
resource "aws_s3_bucket" "thumbnails" {
  bucket = "ablage-thumbnails-${var.environment}"
}

resource "aws_s3_bucket_lifecycle_configuration" "thumbnails" {
  bucket = aws_s3_bucket.thumbnails.id

  rule {
    id     = "delete-old-thumbnails"
    status = "Enabled"

    expiration {
      days = 7
    }
  }
}
```

---

## 🌐 Networking

### VPC Configuration

```hcl
# terraform/modules/networking/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "ablage-vpc-${var.environment}"
  }
}

# Public Subnets (für ALB)
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "ablage-public-${var.availability_zones[count.index]}"
  }
}

# Private Subnets (für Compute)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "ablage-private-${var.availability_zones[count.index]}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "ablage-igw"
  }
}

# NAT Gateways
resource "aws_eip" "nat" {
  count  = var.environment == "prod" ? length(var.availability_zones) : 1
  domain = "vpc"

  tags = {
    Name = "ablage-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = var.environment == "prod" ? length(var.availability_zones) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "ablage-nat-${count.index + 1}"
  }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "ablage-public-rt"
  }
}

resource "aws_route_table" "private" {
  count  = var.environment == "prod" ? length(var.availability_zones) : 1
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "ablage-private-rt-${count.index + 1}"
  }
}
```

---

## 🔐 Ansible Configuration Management

### Inventory Structure

```ini
# ansible/inventory/production.ini

[gpu_backends]
gpu-1.ablage-system.internal ansible_host=10.0.1.10
gpu-2.ablage-system.internal ansible_host=10.0.1.11

[cpu_backends]
cpu-1.ablage-system.internal ansible_host=10.0.2.10
cpu-2.ablage-system.internal ansible_host=10.0.2.11
cpu-3.ablage-system.internal ansible_host=10.0.2.12

[api_servers]
api-1.ablage-system.internal ansible_host=10.0.3.10
api-2.ablage-system.internal ansible_host=10.0.3.11

[all:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
environment=production
```

### GPU Backend Playbook

```yaml
# ansible/playbooks/setup_gpu_backend.yml

---
- name: Setup GPU Backend Servers
  hosts: gpu_backends
  become: yes

  vars:
    cuda_version: "12.1"
    python_version: "3.11"

  tasks:
    - name: Update APT cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install NVIDIA Driver
      apt:
        name:
          - nvidia-driver-535
          - nvidia-utils-535
        state: present
      notify: reboot server

    - name: Install CUDA Toolkit
      block:
        - name: Add NVIDIA CUDA repository
          apt_repository:
            repo: "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /"
            state: present

        - name: Install CUDA
          apt:
            name: cuda-{{ cuda_version }}
            state: present

    - name: Install Python and dependencies
      apt:
        name:
          - python{{ python_version }}
          - python{{ python_version }}-venv
          - python3-pip
          - git
        state: present

    - name: Create application user
      user:
        name: ablage
        shell: /bin/bash
        create_home: yes

    - name: Create application directories
      file:
        path: "{{ item }}"
        state: directory
        owner: ablage
        group: ablage
      loop:
        - /opt/ablage-system
        - /opt/ablage-system/models
        - /var/log/ablage-system

    - name: Install Python packages
      pip:
        name:
          - torch==2.1.0+cu121
          - transformers==4.37.0
          - accelerate==0.25.0
          - fastapi==0.110.0
          - uvicorn[standard]==0.27.0
        extra_args: "--index-url https://download.pytorch.org/whl/cu121"
        virtualenv: /opt/ablage-system/venv
        virtualenv_python: python{{ python_version }}
      become_user: ablage

    - name: Download Hugging Face models
      shell: |
        source /opt/ablage-system/venv/bin/activate
        python -c "
        from transformers import AutoModel, AutoTokenizer
        AutoModel.from_pretrained('deepseek-ai/Janus-Pro-1B', trust_remote_code=True)
        AutoModel.from_pretrained('ucaslcl/GOT-OCR2_0', trust_remote_code=True)
        "
      become_user: ablage
      args:
        creates: /home/ablage/.cache/huggingface

    - name: Configure systemd service
      template:
        src: templates/ablage-gpu-backend.service.j2
        dest: /etc/systemd/system/ablage-gpu-backend.service
      notify: restart ablage-gpu-backend

    - name: Enable and start service
      systemd:
        name: ablage-gpu-backend
        enabled: yes
        state: started

  handlers:
    - name: reboot server
      reboot:
        reboot_timeout: 600

    - name: restart ablage-gpu-backend
      systemd:
        name: ablage-gpu-backend
        state: restarted
```

### CPU Backend Playbook

```yaml
# ansible/playbooks/setup_cpu_backend.yml

---
- name: Setup CPU Backend Servers
  hosts: cpu_backends
  become: yes

  tasks:
    - name: Install system dependencies
      apt:
        name:
          - python3.11
          - python3.11-venv
          - python3-pip
          - build-essential
          - libopencv-dev
        state: present
        update_cache: yes

    - name: Install Surya and Docling
      pip:
        name:
          - surya-ocr>=0.4.0
          - docling>=1.0.0
          - onnxruntime>=1.16.0
          - fastapi==0.110.0
          - uvicorn[standard]==0.27.0
        virtualenv: /opt/ablage-system/venv
        virtualenv_python: python3.11
      become_user: ablage

    - name: Configure systemd service
      template:
        src: templates/ablage-cpu-backend.service.j2
        dest: /etc/systemd/system/ablage-cpu-backend.service
      notify: restart ablage-cpu-backend

  handlers:
    - name: restart ablage-cpu-backend
      systemd:
        name: ablage-cpu-backend
        state: restarted
```

---

## ☸️ Kubernetes Manifests

### Namespace

```yaml
# k8s/base/namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: ablage-system
  labels:
    name: ablage-system
    environment: production
```

### GPU Backend Deployment

```yaml
# k8s/deployments/gpu-backend-deepseek.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-backend-deepseek
  namespace: ablage-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-backend
      model: deepseek
  template:
    metadata:
      labels:
        app: gpu-backend
        model: deepseek
    spec:
      nodeSelector:
        gpu: "true"
        gpu-type: "nvidia-a10g"

      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule

      containers:
        - name: deepseek-backend
          image: ablage-system/gpu-backend-deepseek:latest
          imagePullPolicy: Always

          resources:
            requests:
              memory: "16Gi"
              cpu: "4"
              nvidia.com/gpu: 1
            limits:
              memory: "24Gi"
              cpu: "8"
              nvidia.com/gpu: 1

          env:
            - name: MODEL_NAME
              value: "deepseek-ai/Janus-Pro-1B"
            - name: CUDA_VISIBLE_DEVICES
              value: "0"
            - name: PYTORCH_CUDA_ALLOC_CONF
              value: "max_split_size_mb:512"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url

          ports:
            - containerPort: 8000
              name: http

          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 10

          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10

          volumeMounts:
            - name: model-cache
              mountPath: /root/.cache/huggingface
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: model-cache
          persistentVolumeClaim:
            claimName: huggingface-model-cache
        - name: tmp
          emptyDir:
            sizeLimit: 50Gi
```

### CPU Backend Deployment

```yaml
# k8s/deployments/cpu-backend-surya.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-backend-surya
  namespace: ablage-system
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpu-backend
      model: surya
  template:
    metadata:
      labels:
        app: cpu-backend
        model: surya
    spec:
      containers:
        - name: surya-backend
          image: ablage-system/cpu-backend-surya:latest
          imagePullPolicy: Always

          resources:
            requests:
              memory: "16Gi"
              cpu: "4"
            limits:
              memory: "20Gi"
              cpu: "6"

          env:
            - name: LANGUAGES
              value: "de,en"
            - name: WORKERS
              value: "2"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: url

          ports:
            - containerPort: 8000
              name: http

          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 30

          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa/cpu-backend-hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-backend-hpa
  namespace: ablage-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-backend-surya
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```

### Service

```yaml
# k8s/services/backend-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: ablage-backend
  namespace: ablage-system
spec:
  selector:
    app: gpu-backend
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
  type: ClusterIP
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
---
apiVersion: v1
kind: Service
metadata:
  name: ablage-cpu-backend
  namespace: ablage-system
spec:
  selector:
    app: cpu-backend
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
  type: ClusterIP
```

### Ingress

```yaml
# k8s/ingress/ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ablage-ingress
  namespace: ablage-system
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.ablage-system.com
      secretName: ablage-tls
  rules:
    - host: api.ablage-system.com
      http:
        paths:
          - path: /api/v1/ocr
            pathType: Prefix
            backend:
              service:
                name: ablage-backend
                port:
                  number: 80
          - path: /api/v1/cpu
            pathType: Prefix
            backend:
              service:
                name: ablage-cpu-backend
                port:
                  number: 80
```

### ConfigMap

```yaml
# k8s/config/app-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ablage-system
data:
  LOG_LEVEL: "INFO"
  ENVIRONMENT: "production"
  MAX_UPLOAD_SIZE: "104857600"  # 100MB
  ALLOWED_EXTENSIONS: "pdf,png,jpg,jpeg,tiff"
  PROCESSING_TIMEOUT: "600"
```

### Secrets (Example)

```yaml
# k8s/secrets/database-credentials.yaml (verschlüsselt mit SOPS oder Sealed Secrets)

apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: ablage-system
type: Opaque
stringData:
  url: "postgresql://ablage_admin:REDACTED@ablage-db.cluster.eu-central-1.rds.amazonaws.com:5432/ablage_system"
  username: "ablage_admin"
  password: "REDACTED"
  host: "ablage-db.cluster.eu-central-1.rds.amazonaws.com"
  port: "5432"
  database: "ablage_system"
```

---

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml

name: Build and Deploy

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests
        run: pytest --cov=backend tests/

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./docker/Dockerfile.gpu
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-central-1

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name ablage-system-cluster --region eu-central-1

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/base/
          kubectl apply -f k8s/deployments/
          kubectl rollout status deployment/gpu-backend-deepseek -n ablage-system
          kubectl rollout status deployment/cpu-backend-surya -n ablage-system
```

---

## 📊 Monitoring Infrastructure

### Prometheus

```yaml
# k8s/monitoring/prometheus-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
```

---

## 🔒 Secret Management

### Vault Integration

```hcl
# terraform/modules/secrets/main.tf

resource "vault_mount" "ablage_secrets" {
  path        = "ablage-system"
  type        = "kv-v2"
  description = "Secrets for Ablage System"
}

resource "vault_kv_secret_v2" "database" {
  mount = vault_mount.ablage_secrets.path
  name  = "database"

  data_json = jsonencode({
    username = aws_db_instance.postgres.username
    password = random_password.db_password.result
    host     = aws_db_instance.postgres.address
    port     = aws_db_instance.postgres.port
    database = aws_db_instance.postgres.db_name
  })
}

resource "vault_kv_secret_v2" "api_keys" {
  mount = vault_mount.ablage_secrets.path
  name  = "api-keys"

  data_json = jsonencode({
    huggingface_token = var.huggingface_token
    stripe_api_key    = var.stripe_api_key
  })
}
```

---

## 📝 Best Practices

### 1. State Management
- **Remote Backend**: Immer S3 + DynamoDB für State Locking
- **State Encryption**: Aktiviere Verschlüsselung
- **State Backups**: Automatische Backups konfigurieren

### 2. Modulstruktur
```
terraform/
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   ├── storage/
│   └── monitoring/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── global/
```

### 3. Tagging Strategy
```hcl
default_tags = {
  Project     = "Ablage-System"
  Environment = var.environment
  ManagedBy   = "Terraform"
  CostCenter  = "Engineering"
  Owner       = "platform-team"
}
```

### 4. Security
- Keine Secrets in Code
- Vault/AWS Secrets Manager nutzen
- SOPS für Kubernetes Secrets
- Regelmäßige Security Scans (tfsec, checkov)

### 5. Testing
```bash
# Terraform Validate
terraform validate

# Terraform Plan
terraform plan -out=tfplan

# Cost Estimation
terraform cost estimate tfplan

# Security Scan
tfsec .
checkov -d .
```

---

## 🚀 Deployment Workflow

### Initial Setup
```bash
# 1. Initialize Terraform
cd terraform/environments/prod
terraform init

# 2. Create Workspace
terraform workspace new prod

# 3. Plan Infrastructure
terraform plan -var-file=prod.tfvars

# 4. Apply Infrastructure
terraform apply -var-file=prod.tfvars

# 5. Configure Kubernetes
aws eks update-kubeconfig --name ablage-system-cluster

# 6. Deploy Applications
kubectl apply -f k8s/base/
kubectl apply -f k8s/deployments/

# 7. Run Ansible Configuration
ansible-playbook -i inventory/production.ini playbooks/setup_gpu_backend.yml
```

### Updates
```bash
# 1. Update Code
git pull origin main

# 2. Plan Changes
terraform plan -var-file=prod.tfvars

# 3. Apply Changes
terraform apply -var-file=prod.tfvars

# 4. Rolling Update
kubectl set image deployment/gpu-backend-deepseek \
  deepseek-backend=ablage-system/gpu-backend-deepseek:v2.0

# 5. Monitor Rollout
kubectl rollout status deployment/gpu-backend-deepseek
```

---

## 📚 Weitere Ressourcen

- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [Kubernetes Production Best Practices](https://learnk8s.io/production-best-practices)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
