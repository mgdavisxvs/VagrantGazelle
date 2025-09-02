# Deployment Playbook: VagrantGazelle
**Comprehensive Deployment Strategy & Implementation Guide**  
**Version:** 2.0  
**Target:** Production-Ready Gazelle Tracker Deployment

## Table of Contents
1. [Environment Strategy](#environment-strategy)
2. [Infrastructure-as-Code](#infrastructure-as-code)
3. [Containerization Strategy](#containerization-strategy)
4. [Deployment Pipeline](#deployment-pipeline)
5. [Security Implementation](#security-implementation)
6. [Monitoring & Observability](#monitoring--observability)
7. [Disaster Recovery](#disaster-recovery)

---

## 1. Environment Strategy

### Multi-Environment Architecture

```
Development → Staging → Pre-Production → Production
     ↓           ↓            ↓             ↓
   Local VM   →  Docker   →  K8s Cluster → K8s Cluster
  (Vagrant)     (Compose)   (Single AZ)   (Multi-AZ)
```

### Environment Specifications

#### Development Environment
- **Platform:** Docker Compose (replacing Vagrant)
- **Purpose:** Local development, feature testing
- **Data:** Synthetic/anonymized data
- **Resources:** Minimal (laptop/desktop)

#### Staging Environment
- **Platform:** Kubernetes (single-node)
- **Purpose:** Integration testing, UAT
- **Data:** Production-like test data
- **Resources:** 4 vCPU, 8GB RAM, 100GB storage

#### Pre-Production Environment
- **Platform:** Kubernetes (multi-node)
- **Purpose:** Performance testing, final validation
- **Data:** Production data subset
- **Resources:** 75% of production capacity

#### Production Environment
- **Platform:** Kubernetes (HA cluster)
- **Purpose:** Live application serving
- **Data:** Live production data
- **Resources:** Auto-scaling based on demand

---

## 2. Infrastructure-as-Code

### Terraform Configuration Structure

```
infrastructure/
├── modules/
│   ├── network/
│   ├── security/
│   ├── database/
│   ├── kubernetes/
│   └── monitoring/
├── environments/
│   ├── dev/
│   ├── staging/
│   ├── pre-prod/
│   └── production/
└── shared/
    ├── providers.tf
    ├── backend.tf
    └── variables.tf
```

### Core Infrastructure Components

#### Network Module (`modules/network/main.tf`)
```hcl
# VPC and Networking
resource "aws_vpc" "gazelle_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.environment}-gazelle-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "private_subnets" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.gazelle_vpc.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name        = "${var.environment}-private-subnet-${count.index + 1}"
    Environment = var.environment
  }
}

resource "aws_subnet" "public_subnets" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.gazelle_vpc.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name        = "${var.environment}-public-subnet-${count.index + 1}"
    Environment = var.environment
  }
}
```

#### Kubernetes Module (`modules/kubernetes/main.tf`)
```hcl
# EKS Cluster
resource "aws_eks_cluster" "gazelle_cluster" {
  name     = "${var.environment}-gazelle-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = concat(var.private_subnet_ids, var.public_subnet_ids)
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.cluster_endpoint_public_access_cidrs
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks_encryption.arn
    }
    resources = ["secrets"]
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_cloudwatch_log_group.eks_cluster_logs,
  ]
}

# Node Group
resource "aws_eks_node_group" "gazelle_nodes" {
  cluster_name    = aws_eks_cluster.gazelle_cluster.name
  node_group_name = "${var.environment}-gazelle-nodes"
  node_role_arn   = aws_iam_role.eks_node_group_role.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = var.node_desired_size
    max_size     = var.node_max_size
    min_size     = var.node_min_size
  }

  update_config {
    max_unavailable_percentage = 25
  }

  instance_types = var.node_instance_types
  ami_type       = "AL2_x86_64"
  capacity_type  = "ON_DEMAND"

  remote_access {
    ec2_ssh_key = var.ssh_key_name
  }
}
```

#### Database Module (`modules/database/main.tf`)
```hcl
# RDS MySQL Cluster
resource "aws_rds_cluster" "gazelle_db" {
  cluster_identifier              = "${var.environment}-gazelle-db"
  engine                         = "aurora-mysql"
  engine_version                 = var.mysql_version
  master_username                = var.db_username
  manage_master_user_password    = true
  database_name                  = var.db_name
  
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.gazelle_db_subnet_group.name
  
  backup_retention_period = var.backup_retention_period
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "Sun:04:00-Sun:05:00"
  
  storage_encrypted = true
  kms_key_id       = aws_kms_key.rds_encryption.arn
  
  skip_final_snapshot = var.environment != "production"
  
  enabled_cloudwatch_logs_exports = ["error", "general", "slowquery"]
  
  tags = {
    Name        = "${var.environment}-gazelle-db"
    Environment = var.environment
  }
}

# ElastiCache Redis Cluster
resource "aws_elasticache_replication_group" "gazelle_redis" {
  description            = "Redis cluster for Gazelle ${var.environment}"
  replication_group_id   = "${var.environment}-gazelle-redis"
  
  node_type              = var.redis_node_type
  port                   = 6379
  parameter_group_name   = "default.redis7"
  
  num_cache_clusters     = var.redis_num_cache_nodes
  automatic_failover_enabled = var.environment == "production"
  multi_az_enabled       = var.environment == "production"
  
  subnet_group_name      = aws_elasticache_subnet_group.gazelle_redis_subnet_group.name
  security_group_ids     = [aws_security_group.redis_sg.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                = var.redis_auth_token
  
  tags = {
    Name        = "${var.environment}-gazelle-redis"
    Environment = var.environment
  }
}
```

---

## 3. Containerization Strategy

### Docker Multi-Stage Build

#### Dockerfile
```dockerfile
# Multi-stage build for Gazelle application
FROM php:8.2-fpm-alpine as base

# Install system dependencies
RUN apk add --no-cache \
    nginx \
    mysql-client \
    memcached \
    supervisor \
    curl \
    zip \
    unzip \
    git \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    libmcrypt-dev

# Install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        gd \
        mysqli \
        pdo \
        pdo_mysql \
        opcache \
        bcmath

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Development stage
FROM base as development

# Install Xdebug for development
RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

COPY docker/php/conf.d/20-xdebug.ini /usr/local/etc/php/conf.d/
COPY docker/nginx/nginx.conf /etc/nginx/nginx.conf

WORKDIR /var/www/html
COPY . .

RUN composer install --no-cache --prefer-dist
RUN chown -R www-data:www-data /var/www/html

EXPOSE 80 9000

# Production stage
FROM base as production

# Optimize for production
RUN docker-php-ext-install opcache
COPY docker/php/conf.d/opcache.ini /usr/local/etc/php/conf.d/

# Copy optimized nginx config
COPY docker/nginx/nginx.prod.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

WORKDIR /var/www/html

# Copy application code
COPY --chown=www-data:www-data . .

# Install production dependencies
RUN composer install --no-dev --no-cache --prefer-dist --optimize-autoloader

# Create necessary directories
RUN mkdir -p /var/log/supervisor \
    && mkdir -p /var/run/nginx \
    && mkdir -p /var/cache/nginx

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

# Sphinx search stage
FROM sphinxsearch/sphinx:latest as sphinx

COPY docker/sphinx/sphinx.conf /opt/sphinx/conf/sphinx.conf
RUN chown sphinx:sphinx /opt/sphinx/conf/sphinx.conf

EXPOSE 9312 9306

CMD ["searchd", "--nodetach", "--config", "/opt/sphinx/conf/sphinx.conf"]
```

#### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/var/www/html
      - ./docker/php/conf.d/php.ini:/usr/local/etc/php/conf.d/php.ini
    ports:
      - "8080:80"
    environment:
      - PHP_IDE_CONFIG=serverName=gazelle
      - XDEBUG_CONFIG=remote_host=host.docker.internal
    depends_on:
      - db
      - redis
      - sphinx
    networks:
      - gazelle_network

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
      - ./docker/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"
    networks:
      - gazelle_network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - gazelle_network

  sphinx:
    build:
      context: .
      target: sphinx
    volumes:
      - sphinx_data:/opt/sphinx/data
      - sphinx_logs:/opt/sphinx/log
    ports:
      - "9312:9312"
      - "9306:9306"
    depends_on:
      - db
    networks:
      - gazelle_network

  nginx:
    image: nginx:alpine
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - .:/var/www/html
    ports:
      - "80:80"
    depends_on:
      - app
    networks:
      - gazelle_network

volumes:
  db_data:
  redis_data:
  sphinx_data:
  sphinx_logs:

networks:
  gazelle_network:
    driver: bridge
```

---

## 4. Kubernetes Deployment Manifests

### Namespace and ConfigMap

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gazelle
  labels:
    name: gazelle

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gazelle-config
  namespace: gazelle
data:
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        root /var/www/html;
        index index.php index.html index.htm;
        
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
        
        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
        
        location ~ /\.ht {
            deny all;
        }
    }
  php.ini: |
    memory_limit = 512M
    upload_max_filesize = 100M
    post_max_size = 100M
    max_execution_time = 300
    opcache.enable = 1
    opcache.memory_consumption = 128
```

### Application Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gazelle-app
  namespace: gazelle
  labels:
    app: gazelle-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gazelle-app
  template:
    metadata:
      labels:
        app: gazelle-app
    spec:
      containers:
      - name: gazelle-app
        image: gazelle:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: gazelle-secrets
              key: db-host
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: gazelle-secrets
              key: db-user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gazelle-secrets
              key: db-password
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: gazelle-secrets
              key: redis-host
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gazelle-secrets
              key: redis-password
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
        - name: php-config
          mountPath: /usr/local/etc/php/conf.d
      volumes:
      - name: config-volume
        configMap:
          name: gazelle-config
      - name: php-config
        configMap:
          name: gazelle-config

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gazelle-service
  namespace: gazelle
spec:
  selector:
    app: gazelle-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gazelle-ingress
  namespace: gazelle
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - gazelle.yourdomain.com
    secretName: gazelle-tls
  rules:
  - host: gazelle.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gazelle-service
            port:
              number: 80
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gazelle-hpa
  namespace: gazelle
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gazelle-app
  minReplicas: 3
  maxReplicas: 10
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
```

---

## 5. Deployment Pipeline

### GitLab CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

# Test Stage
unit-tests:
  stage: test
  image: php:8.2-cli
  before_script:
    - apt-get update && apt-get install -y git unzip
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install
  script:
    - vendor/bin/phpunit --coverage-text --colors=never
  artifacts:
    reports:
      junit: tests/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

code-quality:
  stage: test
  image: php:8.2-cli
  before_script:
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer global require squizlabs/php_codesniffer
    - composer global require phpmd/phpmd
  script:
    - ~/.composer/vendor/bin/phpcs --standard=PSR12 src/
    - ~/.composer/vendor/bin/phpmd src/ text cleancode,codesize,controversial,design,naming,unusedcode

# Build Stage
build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --target production -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest

# Security Stage
container-scan:
  stage: security
  image: 
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

secrets-scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --verbose --redact --source="$CI_PROJECT_DIR"

# Staging Deployment
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.gazelle.yourdomain.com
  before_script:
    - kubectl config use-context staging
  script:
    - kubectl set image deployment/gazelle-app gazelle-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n gazelle-staging
    - kubectl rollout status deployment/gazelle-app -n gazelle-staging
  only:
    - develop

# Production Deployment
deploy-production:
  stage: deploy-production
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://gazelle.yourdomain.com
  before_script:
    - kubectl config use-context production
  script:
    - kubectl set image deployment/gazelle-app gazelle-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n gazelle-production
    - kubectl rollout status deployment/gazelle-app -n gazelle-production
  when: manual
  only:
    - master
```

### Deployment Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=${1:-staging}
IMAGE_TAG=${2:-latest}
NAMESPACE="gazelle-${ENVIRONMENT}"

echo "Deploying Gazelle to ${ENVIRONMENT} environment..."

# Apply infrastructure changes
echo "Applying Terraform infrastructure..."
cd infrastructure/environments/${ENVIRONMENT}
terraform plan -out=tfplan
terraform apply tfplan

# Update Kubernetes manifests
echo "Updating Kubernetes manifests..."
cd ../../../k8s
sed -i "s|image: gazelle:.*|image: gazelle:${IMAGE_TAG}|g" deployment.yaml

# Apply Kubernetes resources
echo "Applying Kubernetes resources..."
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml

# Wait for rollout
echo "Waiting for deployment to complete..."
kubectl rollout status deployment/gazelle-app -n ${NAMESPACE}

# Run health checks
echo "Running health checks..."
kubectl run health-check --rm -i --restart=Never --image=curlimages/curl -- \
  curl -f http://gazelle-service.${NAMESPACE}.svc.cluster.local/health

echo "Deployment completed successfully!"
```

---

## 6. Security Implementation

### Secrets Management with HashiCorp Vault

```yaml
# k8s/vault-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gazelle-secrets
  namespace: gazelle
type: Opaque
stringData:
  db-host: "{{ vault "secret/gazelle/db" "host" }}"
  db-user: "{{ vault "secret/gazelle/db" "username" }}"
  db-password: "{{ vault "secret/gazelle/db" "password" }}"
  redis-host: "{{ vault "secret/gazelle/redis" "host" }}"
  redis-password: "{{ vault "secret/gazelle/redis" "password" }}"
  encryption-key: "{{ vault "secret/gazelle/app" "encryption_key" }}"
  site-salt: "{{ vault "secret/gazelle/app" "site_salt" }}"
```

### Network Policies

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gazelle-network-policy
  namespace: gazelle
spec:
  podSelector:
    matchLabels:
      app: gazelle-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: gazelle-app
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

---

This deployment playbook provides a comprehensive strategy for modernizing and deploying the VagrantGazelle application in a production-ready, scalable, and secure manner. The next sections will cover monitoring, CI/CD blueprints, and disaster recovery strategies.