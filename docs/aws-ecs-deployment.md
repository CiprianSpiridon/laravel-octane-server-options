# Deploying Laravel Octane on AWS ECS

This guide provides detailed instructions for deploying Laravel Octane applications with different server options (FrankenPHP, Swoole, and RoadRunner) on Amazon ECS (Elastic Container Service).

## Overview

AWS ECS is a fully managed container orchestration service that makes it easy to deploy, manage, and scale containerized applications. It's an excellent choice for Laravel Octane applications due to:

- **Scalability**: Easily scale your application based on demand
- **High Availability**: Deploy across multiple Availability Zones
- **Integration**: Seamless integration with other AWS services
- **Cost Efficiency**: Only pay for the resources you use

## Prerequisites

Before deploying to ECS, ensure you have:

1. An AWS account with appropriate permissions
2. AWS CLI installed and configured
3. Docker installed locally
4. A Laravel Octane application ready for deployment
5. ECR repository to store your Docker images

## General Deployment Process

Regardless of which Octane server option you choose, the general deployment process follows these steps:

1. Create a Dockerfile for your application
2. Build and push the Docker image to Amazon ECR
3. Create an ECS cluster
4. Define task definitions and services
5. Configure networking and load balancing
6. Deploy and monitor your application

## Server-Specific Configurations

### FrankenPHP on ECS

FrankenPHP's all-in-one nature makes it particularly well-suited for containerized environments like ECS.

#### Dockerfile for FrankenPHP

```dockerfile
FROM dunglas/frankenphp:latest

# Copy application code
COPY . /app
WORKDIR /app

# Install dependencies
RUN composer install --optimize-autoloader --no-dev
RUN php artisan octane:install --server=frankenphp

# Set permissions
RUN chown -R www-data:www-data /app
USER www-data

# Expose ports
EXPOSE 80 443

# Start FrankenPHP with Octane
CMD ["php", "artisan", "octane:start", "--server=frankenphp", "--host=0.0.0.0", "--port=80"]
```

#### ECS Task Definition for FrankenPHP

```json
{
  "family": "laravel-octane-frankenphp",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/laravel-octane-frankenphp:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        },
        {
          "containerPort": 443,
          "hostPort": 443,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "production"
        },
        {
          "name": "APP_KEY",
          "value": "YOUR_APP_KEY"
        },
        {
          "name": "SERVER_NAME",
          "value": "your-domain.com"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laravel-octane-frankenphp",
          "awslogs-region": "REGION",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048"
}
```

#### FrankenPHP ECS Considerations

- **HTTPS Configuration**: FrankenPHP can automatically handle HTTPS, but in ECS you might want to use an Application Load Balancer (ALB) for TLS termination
- **Persistent Storage**: Use EFS if your application needs persistent storage
- **Environment Variables**: Store sensitive environment variables in AWS Secrets Manager or Parameter Store

### Swoole on ECS

Swoole offers excellent performance and is well-suited for high-traffic applications on ECS.

#### Dockerfile for Swoole

```dockerfile
FROM php:8.2-cli

# Install dependencies and Swoole extension
RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    unzip \
    git \
    && docker-php-ext-install zip pdo pdo_mysql \
    && pecl install swoole \
    && docker-php-ext-enable swoole

# Copy application code
COPY . /app
WORKDIR /app

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
RUN composer install --optimize-autoloader --no-dev

# Install Octane
RUN php artisan octane:install --server=swoole

# Set permissions
RUN chown -R www-data:www-data /app
USER www-data

# Expose port
EXPOSE 8000

# Start Swoole server
CMD ["php", "artisan", "octane:start", "--server=swoole", "--host=0.0.0.0", "--port=8000", "--workers=4", "--task-workers=2"]
```

#### ECS Task Definition for Swoole

```json
{
  "family": "laravel-octane-swoole",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/laravel-octane-swoole:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "production"
        },
        {
          "name": "APP_KEY",
          "value": "YOUR_APP_KEY"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laravel-octane-swoole",
          "awslogs-region": "REGION",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "ulimits": [
        {
          "name": "nofile",
          "softLimit": 65536,
          "hardLimit": 65536
        }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048"
}
```

#### Swoole ECS Considerations

- **Worker Configuration**: Adjust the number of workers based on the task's CPU and memory
- **Task Workers**: Configure task workers for handling background processing
- **File Descriptors**: Set appropriate ulimits for file descriptors to handle many concurrent connections
- **Memory Management**: Monitor memory usage and adjust task size accordingly

### RoadRunner on ECS

RoadRunner provides a good balance of performance and compatibility, making it suitable for a wide range of applications on ECS.

#### Dockerfile for RoadRunner

```dockerfile
FROM php:8.2-cli

# Install dependencies
RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    unzip \
    git \
    wget \
    && docker-php-ext-install zip pdo pdo_mysql

# Install RoadRunner
RUN mkdir -p /usr/local/bin \
    && wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.12.1/roadrunner-2.12.1-linux-amd64.tar.gz \
    && tar -xzf roadrunner-2.12.1-linux-amd64.tar.gz \
    && mv roadrunner-2.12.1-linux-amd64/rr /usr/local/bin/rr \
    && chmod +x /usr/local/bin/rr \
    && rm -rf roadrunner-2.12.1-linux-amd64 roadrunner-2.12.1-linux-amd64.tar.gz

# Copy application code
COPY . /app
WORKDIR /app

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
RUN composer install --optimize-autoloader --no-dev

# Install Octane
RUN php artisan octane:install --server=roadrunner

# Set permissions
RUN chown -R www-data:www-data /app
USER www-data

# Expose port
EXPOSE 8000

# Start RoadRunner server
CMD ["/usr/local/bin/rr", "serve", "-c", ".rr.yaml"]
```

#### ECS Task Definition for RoadRunner

```json
{
  "family": "laravel-octane-roadrunner",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/laravel-octane-roadrunner:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "production"
        },
        {
          "name": "APP_KEY",
          "value": "YOUR_APP_KEY"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laravel-octane-roadrunner",
          "awslogs-region": "REGION",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048"
}
```

#### RoadRunner ECS Considerations

- **Binary Version**: Ensure you're using a stable RoadRunner binary version
- **Configuration File**: Properly configure the `.rr.yaml` file for your application
- **Worker Management**: Configure appropriate worker settings based on your task's resources
- **Monitoring**: Set up monitoring for RoadRunner processes

## Setting Up the ECS Infrastructure

### Creating an ECS Cluster

```bash
aws ecs create-cluster --cluster-name laravel-octane-cluster
```

### Creating a Load Balancer

```bash
# Create a security group for the load balancer
aws ec2 create-security-group \
  --group-name laravel-octane-lb-sg \
  --description "Security group for Laravel Octane load balancer" \
  --vpc-id YOUR_VPC_ID

# Allow HTTP and HTTPS traffic
aws ec2 authorize-security-group-ingress \
  --group-id SECURITY_GROUP_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id SECURITY_GROUP_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Create the load balancer
aws elbv2 create-load-balancer \
  --name laravel-octane-lb \
  --subnets SUBNET_ID_1 SUBNET_ID_2 \
  --security-groups SECURITY_GROUP_ID \
  --type application
```

### Creating a Target Group

```bash
aws elbv2 create-target-group \
  --name laravel-octane-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id YOUR_VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30
```

### Creating a Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn LOAD_BALANCER_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=TARGET_GROUP_ARN
```

### Creating an ECS Service

```bash
aws ecs create-service \
  --cluster laravel-octane-cluster \
  --service-name laravel-octane-service \
  --task-definition laravel-octane-SERVERTYPE:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[SUBNET_ID_1,SUBNET_ID_2],securityGroups=[SECURITY_GROUP_ID],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=TARGET_GROUP_ARN,containerName=app,containerPort=8000"
```

## Auto Scaling Configuration

To handle varying loads, set up auto scaling for your ECS service:

```bash
# Create an Auto Scaling Target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/laravel-octane-cluster/laravel-octane-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# Create a CPU Utilization Scaling Policy
aws application-autoscaling put-scaling-policy \
  --policy-name cpu-scaling-policy \
  --service-namespace ecs \
  --resource-id service/laravel-octane-cluster/laravel-octane-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{"TargetValue": 70.0, "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"}}'
```

## Performance Optimization

### Container Insights

Enable Container Insights to monitor your ECS cluster:

```bash
aws ecs update-cluster-settings \
  --cluster laravel-octane-cluster \
  --settings name=containerInsights,value=enabled
```

### X-Ray Integration

For detailed request tracing, integrate AWS X-Ray:

1. Add the X-Ray daemon as a sidecar container in your task definition
2. Install the AWS X-Ray SDK for PHP in your Laravel application
3. Configure the X-Ray middleware in your application

```php
// In app/Providers/AppServiceProvider.php
public function boot()
{
    if (app()->environment('production')) {
        \AWS\XRay\Middleware\Laravel::register();
    }
}
```

### CloudWatch Alarms

Set up CloudWatch alarms to monitor your application:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name laravel-octane-high-cpu \
  --alarm-description "Alarm when CPU exceeds 85%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 60 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ClusterName,Value=laravel-octane-cluster Name=ServiceName,Value=laravel-octane-service \
  --evaluation-periods 3 \
  --alarm-actions SNS_TOPIC_ARN
```

## Server-Specific Optimization Tips

### FrankenPHP Optimization

1. **Caddy Configuration**: Optimize the Caddy configuration for your specific needs
2. **PHP-FPM Settings**: Adjust PHP-FPM settings for optimal performance
3. **Static File Handling**: Configure efficient static file handling

### Swoole Optimization

1. **Worker Count**: Set the optimal number of workers based on your container's CPU cores
2. **Task Workers**: Configure task workers for background processing
3. **Coroutine Settings**: Optimize coroutine settings for your application

### RoadRunner Optimization

1. **Worker Pool**: Configure the worker pool size based on your container's resources
2. **Max Jobs**: Set appropriate max jobs per worker
3. **Timeouts**: Configure proper timeouts for your application

## Deployment Strategies

### Blue/Green Deployment

For zero-downtime deployments, use ECS blue/green deployments with CodeDeploy:

```bash
aws deploy create-application \
  --application-name laravel-octane-app \
  --compute-platform ECS

aws deploy create-deployment-group \
  --application-name laravel-octane-app \
  --deployment-group-name laravel-octane-dg \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --ecs-service-info "clusterName=laravel-octane-cluster,serviceName=laravel-octane-service" \
  --load-balancer-info "targetGroupPairInfoList=[{targetGroups=[{name=laravel-octane-tg-blue},{name=laravel-octane-tg-green}],prodTrafficRoute={listenerArns=[LISTENER_ARN]},testTrafficRoute={listenerArns=[TEST_LISTENER_ARN]}}]" \
  --service-role-arn SERVICE_ROLE_ARN
```

### Canary Deployments

For more cautious deployments, use canary deployments:

1. Create a new task definition with the updated application
2. Update the service with a low percentage of the new task definition
3. Monitor the new tasks for any issues
4. Gradually increase the percentage of new tasks
5. Complete the deployment when confident

## Monitoring and Logging

### CloudWatch Logs

Configure your application to send logs to CloudWatch:

```php
// config/logging.php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['cloudwatch', 'stderr'],
        'ignore_exceptions' => false,
    ],
    'cloudwatch' => [
        'driver' => 'custom',
        'via' => \App\Logging\CloudWatchLoggerFactory::class,
        'level' => 'debug',
    ],
    'stderr' => [
        'driver' => 'monolog',
        'handler' => StreamHandler::class,
        'with' => [
            'stream' => 'php://stderr',
        ],
    ],
],
```

### Prometheus and Grafana

For more detailed monitoring, set up Prometheus and Grafana:

1. Deploy Prometheus as an ECS service
2. Configure your Laravel application to expose metrics
3. Set up Grafana dashboards for visualization

## Cost Optimization

### Fargate Spot Instances

Use Fargate Spot for non-critical workloads:

```bash
aws ecs create-service \
  --cluster laravel-octane-cluster \
  --service-name laravel-octane-spot-service \
  --task-definition laravel-octane-SERVERTYPE:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --capacity-provider-strategy "capacityProvider=FARGATE_SPOT,weight=1" \
  --network-configuration "awsvpcConfiguration={subnets=[SUBNET_ID_1,SUBNET_ID_2],securityGroups=[SECURITY_GROUP_ID],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=TARGET_GROUP_ARN,containerName=app,containerPort=8000"
```

### Reserved Capacity

For predictable workloads, consider purchasing Fargate reserved capacity.

### Right-sizing Tasks

Monitor your application's resource usage and adjust task CPU and memory accordingly.

## Security Best Practices

### IAM Roles

Use IAM roles with least privilege:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "arn:aws:ecr:REGION:ACCOUNT_ID:repository/laravel-octane-SERVERTYPE"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:REGION:ACCOUNT_ID:log-group:/ecs/laravel-octane-SERVERTYPE:*"
    }
  ]
}
```

### Secrets Management

Store sensitive information in AWS Secrets Manager:

```bash
# Create a secret
aws secretsmanager create-secret \
  --name laravel-octane/app-secrets \
  --description "Laravel Octane application secrets" \
  --secret-string '{"APP_KEY":"YOUR_APP_KEY","DB_PASSWORD":"YOUR_DB_PASSWORD"}'
```

Then reference it in your task definition:

```json
"secrets": [
  {
    "name": "APP_KEY",
    "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:laravel-octane/app-secrets:APP_KEY::"
  },
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:laravel-octane/app-secrets:DB_PASSWORD::"
  }
]
```

### VPC Security

Deploy your ECS tasks in a private subnet with a NAT gateway:

```bash
aws ecs create-service \
  --cluster laravel-octane-cluster \
  --service-name laravel-octane-service \
  --task-definition laravel-octane-SERVERTYPE:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[PRIVATE_SUBNET_ID_1,PRIVATE_SUBNET_ID_2],securityGroups=[SECURITY_GROUP_ID],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=TARGET_GROUP_ARN,containerName=app,containerPort=8000"
```

## Troubleshooting

### Common Issues

1. **Task Failures**: Check CloudWatch Logs for application errors
2. **Health Check Failures**: Ensure your application's health check endpoint is working
3. **Networking Issues**: Verify security group and subnet configurations
4. **Resource Constraints**: Check if your tasks have sufficient CPU and memory

### Debugging Tools

1. **ECS Exec**: Use ECS Exec to connect to your running containers:

```bash
aws ecs execute-command \
  --cluster laravel-octane-cluster \
  --task TASK_ID \
  --container app \
  --interactive \
  --command "/bin/bash"
```

2. **CloudWatch Logs Insights**: Use Logs Insights to analyze your application logs:

```
fields @timestamp, @message
| filter @message like /error/i
| sort @timestamp desc
| limit 20
```

## Conclusion

Deploying Laravel Octane applications on AWS ECS provides a scalable, reliable, and cost-effective solution for high-performance PHP applications. Each server option (FrankenPHP, Swoole, and RoadRunner) has its own strengths and considerations when deployed on ECS.

- **FrankenPHP**: Excellent for simplicity and built-in HTTPS support
- **Swoole**: Best for maximum performance and advanced features
- **RoadRunner**: Great balance of performance and compatibility

By following the guidelines in this document, you can successfully deploy, scale, and maintain your Laravel Octane applications on AWS ECS, taking advantage of the cloud's elasticity and reliability while benefiting from Octane's performance improvements. 