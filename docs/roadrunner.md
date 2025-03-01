# RoadRunner In-Depth Guide

RoadRunner is a high-performance PHP application server, load balancer, and process manager written in Go. It's designed to bridge the gap between traditional PHP applications and modern, high-performance server architectures without requiring PHP extensions.

## Key Features

### No PHP Extension Required
- **Standard PHP Installation**: Works with standard PHP installations without custom extensions.
- **Portable Deployment**: Can be deployed in environments where installing extensions is restricted.
- **Simplified Setup**: Easier setup process compared to extension-based solutions.
- **Compatibility**: Works across different PHP versions with minimal configuration changes.

### Go-Based Performance
- **Go Runtime**: Built on the efficient Go programming language for optimal performance.
- **Low Overhead**: Minimal overhead between the server and PHP workers.
- **Efficient Process Management**: Smart worker process management for optimal resource utilization.
- **Lightweight Design**: Designed for minimal resource consumption.

### Compatibility
- **Standard PHP Code**: Works well with traditional PHP coding patterns.
- **Framework Compatibility**: High compatibility with PHP frameworks and libraries.
- **Legacy Application Support**: Can be used to modernize legacy applications.
- **Minimal Code Changes**: Requires fewer code adaptations compared to other high-performance options.

### Memory Reset
- **Clean State**: Resets PHP memory state between requests to prevent memory leaks.
- **Predictable Behavior**: More predictable behavior similar to traditional PHP execution.
- **Resource Cleanup**: Automatic cleanup of resources between requests.
- **Isolation**: Better request isolation compared to some alternatives.

### Plugins System
- **Extendable Architecture**: Plugin system for adding functionality.
- **Middleware Support**: Support for HTTP middleware.
- **Custom Handlers**: Ability to implement custom request handlers.
- **Service Integration**: Easy integration with external services.

## Performance Characteristics

### Speed and Efficiency
- **Request Throughput**: Typically 2-5x faster than traditional PHP-FPM.
- **Latency**: Lower latency, especially under moderate to high load.
- **Worker Management**: Efficient worker process management and recycling.
- **Resource Utilization**: Better CPU and memory utilization than PHP-FPM.

### Memory Management
- **Memory Reset**: Resets PHP memory between requests to prevent leaks.
- **Worker Recycling**: Automatic worker recycling based on configured thresholds.
- **Resource Limits**: Configurable memory and execution time limits.
- **Predictable Usage**: More predictable memory usage patterns.

### Concurrency Model
- **Process-Based Concurrency**: Uses multiple PHP worker processes for concurrency.
- **Load Balancing**: Built-in load balancing between workers.
- **Queue Management**: Smart request queuing during high load.
- **Backpressure Handling**: Mechanisms to handle backpressure under high load.

## Detailed Use Cases

### Shared Hosting Environments
RoadRunner excels in environments where installing PHP extensions is not possible:

- **Restricted Hosting**: Environments with restrictions on PHP extension installation.
- **Managed Hosting**: Managed hosting platforms with limited customization options.
- **Security-Restricted Environments**: Environments with strict security policies.
- **Containerized Deployments**: Simplified container images without custom extensions.

```yaml
# .rr.yaml for shared hosting environments
server:
  command: "php worker.php"
  relay: "pipes"

http:
  address: "127.0.0.1:8080"
  pool:
    num_workers: 4
    max_jobs: 64
    allocate_timeout: 60s
    destroy_timeout: 60s
```

### Legacy Application Modernization
RoadRunner provides a path to modernize legacy applications without major rewrites:

- **PHP 5.x Applications**: Older applications that need performance improvements.
- **Monolithic Applications**: Large monolithic applications that are difficult to refactor.
- **Gradual Migration**: Applications being gradually migrated to modern architectures.
- **Technical Debt Reduction**: Improving performance while addressing technical debt.

```php
// Example of adapting a legacy application to use RoadRunner
// worker.php

// Include the RoadRunner PSR-7 worker
use Spiral\RoadRunner\Worker;
use Nyholm\Psr7\Factory\Psr17Factory;
use Spiral\RoadRunner\Http\PSR7Worker;

// Legacy application bootstrap (with modifications)
require_once 'legacy_bootstrap.php';

// Create PSR-7 worker
$worker = Worker::create();
$factory = new Psr17Factory();
$psr7 = new PSR7Worker($worker, $factory, $factory, $factory);

while ($request = $psr7->waitRequest()) {
    try {
        // Capture output
        ob_start();
        
        // Set up legacy environment variables
        $_SERVER = $request->getServerParams();
        $_GET = $request->getQueryParams();
        $_POST = $request->getParsedBody() ?? [];
        $_COOKIE = $request->getCookieParams();
        
        // Include the legacy front controller
        include 'legacy_index.php';
        
        // Capture output and send response
        $content = ob_get_clean();
        $response = $factory->createResponse(200)->withBody(
            $factory->createStream($content)
        );
        
        $psr7->respond($response);
    } catch (\Throwable $e) {
        // Handle errors
        $psr7->respond(
            $factory->createResponse(500)->withBody(
                $factory->createStream('Internal Server Error: ' . $e->getMessage())
            )
        );
    }
}
```

### Consistent Development and Production Environments
RoadRunner helps maintain consistency between different environments:

- **Development Parity**: Maintain the same server architecture in development and production.
- **CI/CD Pipelines**: Consistent testing and deployment environments.
- **Cross-Platform Development**: Teams working across different operating systems.
- **Reproducible Environments**: Easily reproducible server environments.

```yaml
# docker-compose.yml for consistent environments

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=development
      - DB_HOST=database
      - REDIS_HOST=redis
    depends_on:
      - database
      - redis

  database:
    image: mysql:8.0
    environment:
      - MYSQL_DATABASE=app
      - MYSQL_USER=user
      - MYSQL_PASSWORD=password
      - MYSQL_ROOT_PASSWORD=rootpassword
    volumes:
      - db_data:/var/lib/mysql

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

volumes:
  db_data:
  redis_data:
```

### Standard PHP Applications
RoadRunner is ideal for standard PHP applications that need performance improvements:

- **Content Management Systems**: WordPress, Drupal, or custom CMS platforms.
- **E-commerce Platforms**: Online stores and marketplaces.
- **Enterprise Applications**: Internal business applications.
- **API Services**: REST or GraphQL API services.

```php
// Example of optimizing a standard Laravel application for RoadRunner
// routes/web.php

// Pre-resolve frequently used services
$userRepository = app(App\Repositories\UserRepository::class);
$productService = app(App\Services\ProductService::class);

Route::get('/products', function () use ($productService) {
    return $productService->getAllProducts();
});

Route::get('/users', function () use ($userRepository) {
    return $userRepository->getAllUsers();
});

// worker.php
use Spiral\RoadRunner\Worker;
use Spiral\RoadRunner\Http\PSR7Worker;
use Nyholm\Psr7\Factory\Psr17Factory;
use Laravel\Octane\RoadRunner\ServerProcessInspector;
use Laravel\Octane\RoadRunner\ServerStateFile;

$app = require __DIR__.'/bootstrap/app.php';

$worker = Worker::create();
$factory = new Psr17Factory();
$psr7 = new PSR7Worker($worker, $factory, $factory, $factory);

$inspector = new ServerProcessInspector(
    new ServerStateFile(storage_path('logs/octane-server-state.json'))
);

while ($request = $psr7->waitRequest()) {
    try {
        $response = $app->handle($request);
        $psr7->respond($response);
        
        // Reset Laravel application state
        $app->make('db')->reconnect();
        $app->make('cache')->driver()->reset();
        $app->make('log')->reset();
        
        // Clear resolved instances
        $app->forgetScopedInstances();
        $app->forgetInstancesForgetScopedInstances();
    } catch (\Throwable $e) {
        $psr7->respond(
            $factory->createResponse(500)->withBody(
                $factory->createStream('Internal Server Error: ' . $e->getMessage())
            )
        );
    }
}
```

### Restricted Environments
RoadRunner is well-suited for environments with strict software policies:

- **Government Applications**: Applications in government agencies with strict software policies.
- **Financial Services**: Banking and financial applications with security restrictions.
- **Healthcare Systems**: Medical and healthcare applications with compliance requirements.
- **Enterprise Environments**: Corporate environments with strict IT policies.

```yaml
# .rr.yaml for restricted environments with security considerations
server:
  command: "php worker.php"
  relay: "pipes"
  env:
    - SECURE_MODE=1
    - RESTRICTED_FUNCTIONS=exec,shell_exec,system,passthru

http:
  address: "127.0.0.1:8080"
  middleware: ["headers", "gzip"]
  pool:
    num_workers: 8
    max_jobs: 1000
    supervisor:
      exec_ttl: 60s
      
  headers:
    response:
      X-Content-Type-Options: nosniff
      X-Frame-Options: DENY
      Content-Security-Policy: "default-src 'self'"
      Strict-Transport-Security: "max-age=31536000; includeSubDomains"
```

## Laravel Octane Integration

### Installation
```bash
# Install Laravel Octane
composer require laravel/octane

# Install RoadRunner for Octane
php artisan octane:install --server=roadrunner
```

### Configuration
```php
// config/octane.php
return [
    'server' => 'roadrunner',
    'roadrunner' => [
        'http' => [
            'address' => '0.0.0.0:8000',
            'max_request_size' => '32MB',
        ],
        'server' => [
            'command' => 'php artisan octane:start',
        ],
        'pool' => [
            'num_workers' => 8,
            'max_jobs' => 500,
            'allocation_strategy' => 'auto',
        ],
    ],
    'warm' => [
        // Classes to pre-resolve in the container
        App\Providers\RouteServiceProvider::class,
        // Common services
        App\Services\UserService::class,
    ],
];
```

### Starting the Server
```bash
php artisan octane:start --server=roadrunner --host=0.0.0.0 --port=8000 --workers=8
```

### Production Deployment
```bash
# Start RoadRunner in production with Supervisor
[program:laravel-octane]
process_name=%(program_name)s_%(process_num)02d
command=/path/to/your/app/rr serve -c /path/to/your/app/.rr.yaml
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/path/to/your/app/storage/logs/octane.log
stopwaitsecs=60
```

## Limitations and Considerations

### Performance Ceiling
- **Raw Performance**: Generally slower than Swoole at peak performance.
- **Feature Set**: Fewer advanced features compared to Swoole.
- **Concurrency Model**: Less sophisticated concurrency model than Swoole's coroutines.
- **WebSocket Support**: Less mature WebSocket support compared to Swoole.

### Separate Binary
- **Binary Installation**: Requires installation of the RoadRunner binary.
- **Version Management**: Need to manage RoadRunner binary versions.
- **Deployment Complexity**: Additional step in deployment processes.
- **Update Management**: Need to update the binary separately from PHP.

```bash
# Installing RoadRunner binary
curl -s https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/install.sh | bash
```

### Process Management Complexity
- **Worker Configuration**: More complex worker configuration compared to traditional setups.
- **Resource Allocation**: Need to carefully configure worker resources.
- **Monitoring**: Different approach to monitoring worker processes.
- **Debugging**: More complex debugging of worker processes.

```yaml
# Advanced worker configuration in .rr.yaml
server:
  command: "php worker.php"
  relay: "pipes"

http:
  address: "0.0.0.0:8080"
  pool:
    num_workers: 8
    max_jobs: 1000
    allocate_timeout: 60s
    destroy_timeout: 60s
    supervisor:
      exec_ttl: 60s
      max_worker_memory: 128
      max_worker_idle_time: 10s
```

### Documentation and Community
- **Documentation Gaps**: Less comprehensive documentation compared to Swoole.
- **Community Size**: Smaller community compared to more established options.
- **Learning Resources**: Fewer tutorials and learning resources.
- **Third-Party Integrations**: Fewer third-party integrations and plugins.

## Real-World Examples

### Government Agency Portal
A government agency migrated their citizen portal to RoadRunner:
- 3x improvement in response times
- Compliance with strict security policies that prohibited PHP extensions
- Simplified deployment in restricted environments
- Reduced server costs by 40%

### Legacy Banking Application
A financial institution modernized their legacy PHP application with RoadRunner:
- Improved transaction processing speed by 200%
- Maintained compatibility with existing codebase
- Met strict security and compliance requirements
- Gradual migration path without service disruption

### Healthcare Records System
A healthcare provider implemented RoadRunner for their patient records system:
- Improved record retrieval times by 60%
- Maintained HIPAA compliance in restricted environments
- Simplified deployment across multiple facilities
- Reduced infrastructure costs while improving performance

### Enterprise Content Management
A large corporation used RoadRunner for their document management system:
- Handled 2x more concurrent users with the same infrastructure
- Simplified deployment across diverse corporate environments
- Maintained compatibility with legacy integrations
- Improved document processing and search performance

## Conclusion

RoadRunner represents a balanced approach to PHP application performance, offering significant improvements over traditional PHP-FPM setups while maintaining high compatibility and flexibility. Its integration with Laravel Octane provides a straightforward path to performance improvements without requiring PHP extensions.

While it may not offer the raw performance capabilities of Swoole, RoadRunner's focus on compatibility, portability, and standard PHP practices makes it an excellent choice for teams looking to modernize their applications with minimal disruption. It's particularly well-suited for environments with restrictions on PHP extensions or for applications where maintaining compatibility with existing code is a priority.

The trade-offs in peak performance are often offset by the simplified deployment, better compatibility, and more familiar development model, making RoadRunner an excellent choice for many PHP applications seeking performance improvements without radical architecture changes. 