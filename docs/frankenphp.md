# FrankenPHP In-Depth Guide

FrankenPHP is a modern, all-in-one solution for running PHP applications with high performance. It's built on top of the Caddy web server and integrates PHP directly, eliminating the need for separate PHP-FPM and web server setups.

## Key Features

### All-in-One Solution
- **Integrated PHP and Web Server**: FrankenPHP combines PHP with Caddy server, eliminating the traditional separation between web server and PHP-FPM.
- **Simplified Architecture**: Reduces the number of components in your stack, leading to easier maintenance and troubleshooting.
- **Unified Configuration**: Single configuration file for both web server and PHP settings.

### Built-in HTTPS
- **Automatic Certificate Management**: Leverages Caddy's automatic HTTPS capabilities to generate and renew SSL certificates.
- **Zero Configuration SSL**: Get HTTPS working out of the box without manual certificate management.
- **Let's Encrypt Integration**: Seamless integration with Let's Encrypt for free, valid SSL certificates.

### Modern Protocol Support
- **HTTP/3 Support**: Native support for the latest HTTP protocol version.
- **HTTP/2 Push**: Ability to push resources to clients before they request them.
- **WebSocket Support**: Built-in support for WebSocket connections.

### Docker-Friendly
- **Official Docker Images**: Well-maintained Docker images ready for production use.
- **Minimal Image Size**: Optimized container images with small footprints.
- **Docker Compose Examples**: Ready-to-use examples for Docker Compose deployments.

### Worker Management
- **Efficient Process Handling**: Smart worker process management for optimal resource utilization.
- **Auto-Scaling**: Automatically adjusts the number of workers based on load.
- **Graceful Reloads**: Zero-downtime reloads when updating your application.

## Performance Characteristics

### Memory Efficiency
- **Shared Memory Model**: Efficient memory usage through shared memory between workers.
- **Lower Memory Footprint**: Typically uses less memory than traditional PHP-FPM setups.
- **Memory Limit Controls**: Fine-grained control over PHP memory limits per worker.

### Request Handling
- **Fast Request Processing**: Direct handling of HTTP requests without additional layers.
- **Connection Pooling**: Reuses connections for better performance.
- **Static File Acceleration**: Efficient serving of static files through Caddy.

### Concurrency
- **Worker-Based Concurrency**: Handles concurrent requests through multiple worker processes.
- **Connection Limits**: Configurable limits for concurrent connections.
- **Request Queuing**: Smart queuing of requests during high load.

## Detailed Use Cases

### Microservices Architecture
FrankenPHP excels in microservices environments due to its lightweight nature and container-friendliness:

- **Independent Service Deployment**: Each microservice can be packaged with its own FrankenPHP instance.
- **Resource Efficiency**: Lower resource overhead compared to traditional setups makes it ideal for numerous small services.
- **Simplified Networking**: Built-in service discovery and routing capabilities through Caddy.

```yaml
# docker-compose.yml example for a microservice
version: '3'

services:
  auth-service:
    image: dunglas/frankenphp
    volumes:
      - ./auth-service:/app
    environment:
      - SERVER_NAME=auth.example.com
      - PHP_MEMORY_LIMIT=256M
    ports:
      - "8001:80"
      - "8443:443"

  user-service:
    image: dunglas/frankenphp
    volumes:
      - ./user-service:/app
    environment:
      - SERVER_NAME=users.example.com
      - PHP_MEMORY_LIMIT=256M
    ports:
      - "8002:80"
      - "8444:443"
```

### Docker and Kubernetes Environments
FrankenPHP provides excellent integration with container orchestration platforms:

- **Kubernetes-Ready**: Works seamlessly with Kubernetes for orchestration.
- **Health Checks**: Built-in health check endpoints for container orchestration.
- **Resource Constraints**: Works well with memory and CPU limits in containerized environments.

```yaml
# kubernetes deployment example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel-app
  template:
    metadata:
      labels:
        app: laravel-app
    spec:
      containers:
      - name: laravel-app
        image: dunglas/frankenphp
        ports:
        - containerPort: 80
        - containerPort: 443
        volumeMounts:
        - name: app-code
          mountPath: /app
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
      volumes:
      - name: app-code
        persistentVolumeClaim:
          claimName: app-code-pvc
```

### New Projects Without Legacy Constraints
For greenfield projects, FrankenPHP offers several advantages:

- **Modern Architecture**: Start with a modern, efficient architecture from day one.
- **Future-Proof**: Built with modern PHP practices and HTTP standards in mind.
- **Developer Experience**: Simplified local development with automatic HTTPS.

### Developer Environments
FrankenPHP shines in local development scenarios:

- **Quick Setup**: Fast setup with minimal configuration.
- **Local HTTPS**: Develop locally with HTTPS without certificate hassles.
- **Hot Reloading**: Support for hot reloading during development.

```bash
# Quick local development setup
docker run -v $PWD:/app -p 80:80 -p 443:443 dunglas/frankenphp
```

### Simple Deployment Needs
For teams wanting to minimize operational complexity:

- **Reduced Infrastructure**: Fewer components to manage and monitor.
- **Simplified Logging**: Consolidated logs from a single service.
- **Easier Troubleshooting**: Less moving parts means easier problem diagnosis.

## Laravel Octane Integration

### Installation
```bash
# Install Laravel Octane
composer require laravel/octane

# Install FrankenPHP
php artisan octane:install --server=frankenphp
```

### Configuration
```php
// config/octane.php
return [
    'server' => 'frankenphp',
    'https' => true,
    'listeners' => [
        'task' => [
            'timeout' => 120,
            'threads' => 16,
        ],
    ],
    'warm' => [
        // Classes to pre-resolve in the container
        App\Providers\RouteServiceProvider::class,
    ],
];
```

### Starting the Server
```bash
php artisan octane:start --server=frankenphp --host=0.0.0.0 --port=80
```

### Docker Deployment
```dockerfile
FROM dunglas/frankenphp

COPY . /app
WORKDIR /app

RUN composer install --optimize-autoloader --no-dev
RUN php artisan octane:install --server=frankenphp

EXPOSE 80 443

CMD ["php", "artisan", "octane:start", "--server=frankenphp", "--host=0.0.0.0", "--port=80"]
```

## Limitations and Considerations

### Maturity
- **Newer Technology**: Less battle-tested in large-scale production environments compared to more established options.
- **Evolving API**: The API and configuration options may change as the project matures.
- **Community Size**: Smaller community compared to traditional setups or Swoole.

### Learning Curve
- **New Configuration Patterns**: Different configuration approach compared to traditional Nginx/Apache + PHP-FPM setups.
- **Caddy Knowledge**: Beneficial to understand Caddy server configuration for advanced use cases.
- **Troubleshooting Resources**: Fewer resources available for troubleshooting complex issues.

### Extension Compatibility
- **PHP Extension Support**: Some PHP extensions may have compatibility issues.
- **Extension Loading**: Different approach to loading and configuring PHP extensions.
- **Third-Party Libraries**: Some libraries assuming traditional PHP execution model may have issues.

### Debugging
- **Debugging Tools**: Fewer specialized debugging tools compared to traditional setups.
- **Profiling**: Different approach needed for application profiling.
- **Monitoring**: May require adapting monitoring solutions for the combined server model.

## Real-World Examples

### E-commerce Platform
An e-commerce platform migrated from traditional Nginx + PHP-FPM to FrankenPHP and experienced:
- 40% reduction in response times
- 30% lower server costs due to better resource utilization
- Simplified deployment pipeline with Docker containers

### Content Management System
A content-heavy CMS implemented with FrankenPHP achieved:
- Improved content delivery speeds
- Simplified SSL certificate management
- Reduced DevOps overhead for the small team

### API Gateway
A company using FrankenPHP as an API gateway for their microservices architecture reported:
- Simplified service mesh implementation
- Reduced latency between services
- Easier deployment and scaling of individual services

## Conclusion

FrankenPHP represents a modern approach to PHP application deployment, particularly well-suited for containerized environments and new projects. While it may not have the raw performance capabilities of Swoole or the compatibility advantages of RoadRunner, it offers an excellent balance of features, simplicity, and performance that makes it an attractive option for many use cases.

Its integration with Laravel Octane provides a straightforward path to significant performance improvements with minimal configuration, making it an excellent choice for teams looking to modernize their Laravel applications without a steep learning curve. 