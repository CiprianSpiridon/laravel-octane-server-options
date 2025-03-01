# Migration Strategies for Laravel Octane

This document outlines strategies for migrating existing Laravel applications to Laravel Octane with different server options. A successful migration requires careful planning, testing, and implementation to ensure a smooth transition with minimal disruption.

## General Migration Process

Regardless of which Octane server option you choose, the migration process follows these general steps:

### 1. Assessment and Preparation

- **Application Audit**: Review your codebase for potential compatibility issues:
  - Identify singletons and static variables that might persist between requests
  - Check for code that assumes a fresh application state for each request
  - Review resource management (file handles, database connections, etc.)
  - Identify third-party packages that might have compatibility issues

- **Performance Baseline**: Establish current performance metrics to measure improvements:
  - Response times for key routes
  - Throughput under various load conditions
  - Memory and CPU utilization
  - Database connection usage

- **Environment Setup**: Prepare a testing environment that mirrors production:
  - Same PHP version and extensions
  - Similar hardware resources
  - Comparable database size and configuration
  - Representative traffic patterns

### 2. Initial Implementation

- **Install Laravel Octane**:
  ```bash
  composer require laravel/octane
  ```

- **Install Server Dependencies**:
  - For FrankenPHP:
    ```bash
    php artisan octane:install --server=frankenphp
    ```
  
  - For Swoole:
    ```bash
    pecl install swoole
    php artisan octane:install --server=swoole
    ```
  
  - For RoadRunner:
    ```bash
    php artisan octane:install --server=roadrunner
    ```

- **Basic Configuration**:
  - Configure `config/octane.php` with appropriate settings
  - Set up server-specific configuration files

### 3. Code Adaptation

- **Fix State Persistence Issues**:
  ```php
  // Before: Problematic static property
  class UserCounter
  {
      public static $count = 0;
  }
  
  // After: Reset between requests
  class UserCounter
  {
      public static $count = 0;
      
      public static function reset()
      {
          self::$count = 0;
      }
  }
  
  // In AppServiceProvider
  public function register()
  {
      $this->app->terminating(function () {
          UserCounter::reset();
      });
  }
  ```

- **Implement Resource Cleanup**:
  ```php
  // In AppServiceProvider
  public function register()
  {
      $this->app->terminating(function ($app) {
          // Reset database connections
          $app->make('db')->reconnect();
          
          // Clear cached instances
          $app->forgetScopedInstances();
          
          // Reset other services as needed
          $app->make('cache')->driver()->reset();
          $app->make('log')->reset();
      });
  }
  ```

- **Adapt Third-Party Packages**:
  - Fork and modify incompatible packages if necessary
  - Implement workarounds for specific issues
  - Consider alternative packages if adaptation is too complex

### 4. Testing and Validation

- **Functional Testing**:
  - Run your full test suite to ensure functionality is preserved
  - Test edge cases and complex workflows
  - Verify file uploads, downloads, and other I/O operations

- **Performance Testing**:
  - Benchmark key routes and operations
  - Test under various load conditions
  - Compare with baseline metrics

- **Stability Testing**:
  - Run long-duration tests to identify memory leaks
  - Test recovery from errors and exceptions
  - Verify proper cleanup between requests

### 5. Deployment Strategy

- **Phased Rollout**:
  - Deploy to staging environment first
  - Consider a canary deployment for production
  - Implement feature flags to enable/disable Octane if needed

- **Monitoring Setup**:
  - Configure appropriate monitoring for the new server architecture
  - Set up alerts for key metrics
  - Implement logging for Octane-specific issues

- **Rollback Plan**:
  - Document steps to revert to PHP-FPM if issues arise
  - Ensure configuration backups are available
  - Test the rollback process before production deployment

## Server-Specific Migration Strategies

### Migrating to FrankenPHP

#### Preparation

- **Docker Integration**:
  - If using Docker, update your Dockerfile:
    ```dockerfile
    FROM dunglas/frankenphp
    
    COPY . /app
    WORKDIR /app
    
    RUN composer install --optimize-autoloader --no-dev
    
    EXPOSE 80 443
    
    CMD ["php", "artisan", "octane:start", "--server=frankenphp", "--host=0.0.0.0", "--port=80"]
    ```

- **Web Server Configuration**:
  - If migrating from Nginx or Apache, adapt your web server configuration to FrankenPHP's Caddy-based approach
  - Convert rewrite rules and custom headers
  - Migrate SSL certificate configuration to Caddy format

#### Implementation

- **Configuration**:
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
          // Pre-resolve frequently used services
          App\Providers\RouteServiceProvider::class,
      ],
  ];
  ```

- **Caddy Configuration**:
  - Create a `Caddyfile` for advanced configuration:
    ```
    {
        auto_https off
        admin off
    }
    
    :80 {
        root * public/
        php_server {
            resolve_root_symlink
        }
        encode gzip
        file_server
    }
    ```

#### Special Considerations

- **HTTPS Setup**:
  - Take advantage of automatic HTTPS by configuring domains properly
  - Ensure DNS is properly configured for certificate issuance

- **Static File Handling**:
  - Optimize static file serving through Caddy
  - Configure appropriate caching headers

- **Container Orchestration**:
  - Update Kubernetes or Docker Compose configurations
  - Configure health checks and readiness probes

### Migrating to Swoole

#### Preparation

- **Extension Installation**:
  - Install the Swoole extension:
    ```bash
    pecl install swoole
    echo "extension=swoole.so" > /etc/php/8.x/cli/conf.d/swoole.ini
    ```

- **Dependency Review**:
  - Identify packages that might conflict with Swoole
  - Test each third-party package for compatibility
  - Prepare alternatives or workarounds for incompatible packages

#### Implementation

- **Configuration**:
  ```php
  // config/octane.php
  return [
      'server' => 'swoole',
      'swoole' => [
          'options' => [
              'log_file' => storage_path('logs/swoole_http.log'),
              'log_level' => 4,
              'worker_num' => 8,
              'task_worker_num' => 4,
              'task_enable_coroutine' => true,
              'max_request' => 1000,
              'open_tcp_nodelay' => true,
              'open_http2_protocol' => true,
          ],
      ],
      'warm' => [
          // Pre-resolve frequently used services
          App\Providers\RouteServiceProvider::class,
      ],
  ];
  ```

- **State Management**:
  - Implement more thorough state reset between requests:
    ```php
    // In AppServiceProvider
    public function register()
    {
        $this->app->terminating(function ($app) {
            // Reset database connections
            $app->make('db')->reconnect();
            
            // Reset services
            $app->make('auth')->forgetGuards();
            $app->make('cache')->driver()->reset();
            $app->make('log')->reset();
            $app->make('session')->driver()->reset();
            
            // Clear resolved instances
            $app->forgetScopedInstances();
            foreach ($this->resetServices as $service) {
                $app->forgetInstance($service);
            }
            
            // Reset static properties
            foreach ($this->resetStaticProperties as $class) {
                $class::reset();
            }
        });
    }
    ```

#### Special Considerations

- **Coroutine Usage**:
  - Consider refactoring I/O-heavy operations to use coroutines:
    ```php
    use Swoole\Coroutine;
    
    // In a controller or service
    public function fetchMultipleApis()
    {
        $results = [];
        
        Coroutine\run(function() use (&$results) {
            $results['users'] = Coroutine::create(function() {
                return Http::get('https://api.example.com/users')->json();
            });
            
            $results['products'] = Coroutine::create(function() {
                return Http::get('https://api.example.com/products')->json();
            });
            
            $results['orders'] = Coroutine::create(function() {
                return Http::get('https://api.example.com/orders')->json();
            });
        });
        
        return $results;
    }
    ```

- **WebSocket Implementation**:
  - If using WebSockets, migrate to Swoole's native WebSocket server:
    ```php
    // In a service provider
    Octane::webSocketHandler(function ($request, $websocket) {
        // Authenticate the WebSocket connection
        if (!Auth::check()) {
            return $websocket->close();
        }
        
        // Join user to appropriate channels
        $user = Auth::user();
        $websocket->join("user.{$user->id}");
        
        return $websocket;
    });
    
    Octane::webSocketMessage(function ($message, $websocket) {
        $payload = json_decode($message, true);
        
        // Process message
        // ...
        
        // Broadcast response
        $websocket->broadcast(json_encode([
            'type' => 'response',
            'data' => $responseData,
        ]));
    });
    ```

- **Task Workers**:
  - Offload CPU-intensive tasks to Swoole task workers:
    ```php
    // In a controller
    public function processLargeReport(Request $request)
    {
        $reportId = $request->input('report_id');
        $parameters = $request->input('parameters');
        
        // Queue report generation in task worker
        Octane::task(function () use ($reportId, $parameters) {
            $reportGenerator = new ReportGenerator();
            $result = $reportGenerator->generate($reportId, $parameters);
            
            // Store result or notify user
            ReportResult::create([
                'report_id' => $reportId,
                'result' => $result,
                'completed_at' => now(),
            ]);
            
            // Notify user
            event(new ReportCompleted($reportId));
        });
        
        return response()->json([
            'status' => 'processing',
            'report_id' => $reportId,
        ]);
    }
    ```

### Migrating to RoadRunner

#### Preparation

- **Binary Installation**:
  - Install the RoadRunner binary:
    ```bash
    curl -s https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/install.sh | bash
    ```

- **Configuration Files**:
  - Create the RoadRunner configuration file:
    ```yaml
    # .rr.yaml
    server:
      command: "php artisan octane:start --server=roadrunner"
      relay: "pipes"
    
    http:
      address: "0.0.0.0:8080"
      pool:
        num_workers: 8
        max_jobs: 1000
        allocate_timeout: 60s
        destroy_timeout: 60s
    ```

#### Implementation

- **Configuration**:
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
          // Pre-resolve frequently used services
          App\Providers\RouteServiceProvider::class,
      ],
  ];
  ```

- **Worker Script**:
  - The worker script is automatically generated by Octane, but you may need to customize it for complex applications

#### Special Considerations

- **File Uploads**:
  - Test file upload functionality thoroughly
  - Adjust `max_request_size` if needed for large uploads

- **Session Handling**:
  - Verify session persistence and cleanup
  - Consider using Redis for session storage

- **Long-Running Processes**:
  - Implement proper timeout handling for long-running processes
  - Consider using queue workers for very long operations

## Handling Common Migration Challenges

### Memory Leaks

Memory leaks are one of the most common issues when migrating to persistent PHP applications:

1. **Identify Leaks**:
   - Use tools like Blackfire or Tideways to profile memory usage
   - Implement memory usage logging between requests
   - Look for gradually increasing memory usage over time

2. **Common Causes**:
   - Appending to arrays or collections without bounds
   - Not cleaning up event listeners
   - Caching data in static properties
   - Circular references preventing garbage collection

3. **Solutions**:
   - Implement explicit cleanup in the `terminating` callback
   - Set limits on collection sizes
   - Use weak references for caches
   - Reset static properties between requests

### Database Connection Management

Persistent connections can cause issues if not properly managed:

1. **Connection Pooling**:
   - Configure appropriate connection pool size
   - Implement connection health checks
   - Handle connection timeouts gracefully

2. **Transaction Cleanup**:
   - Ensure transactions are properly committed or rolled back
   - Implement transaction timeout handling
   - Add safeguards to prevent orphaned transactions

3. **Solutions**:
   ```php
   // In AppServiceProvider
   public function register()
   {
       $this->app->terminating(function ($app) {
           // Reset database connections
           $db = $app->make('db');
           
           // Rollback any uncommitted transactions
           foreach ($db->getConnections() as $connection) {
               if ($connection->transactionLevel() > 0) {
                   $connection->rollBack(0);
               }
           }
           
           // Reconnect to reset any stale connections
           $db->reconnect();
       });
   }
   ```

### Third-Party Package Compatibility

Many packages assume a traditional PHP execution model:

1. **Identify Problematic Packages**:
   - Test each package individually
   - Look for packages that use global state
   - Check for packages that assume request isolation

2. **Solutions**:
   - Fork and modify incompatible packages
   - Implement wrapper classes to reset state
   - Use dependency injection instead of global access
   - Contact package maintainers about Octane compatibility

3. **Example Wrapper**:
   ```php
   // Wrapper for a problematic package
   class PackageWrapper
   {
       private $instance;
       
       public function __construct()
       {
           $this->instance = new ProblemPackage();
       }
       
       public function __call($method, $args)
       {
           return $this->instance->$method(...$args);
       }
       
       public function reset()
       {
           $this->instance = new ProblemPackage();
       }
   }
   
   // In a service provider
   $this->app->singleton('problem-package', function () {
       return new PackageWrapper();
   });
   
   $this->app->terminating(function ($app) {
       $app->make('problem-package')->reset();
   });
   ```

## Production Deployment Strategies

### Blue-Green Deployment

Blue-green deployment minimizes downtime and risk:

1. **Setup**:
   - Maintain two identical production environments (Blue and Green)
   - Only one environment is live at any time

2. **Process**:
   - Deploy Octane to the inactive environment
   - Test thoroughly
   - Switch traffic to the new environment
   - Keep the old environment as a fallback

3. **Benefits**:
   - Zero downtime deployment
   - Easy rollback if issues arise
   - Opportunity for final validation in a production-identical environment

### Canary Deployment

Canary deployment gradually shifts traffic to the new system:

1. **Setup**:
   - Deploy both PHP-FPM and Octane versions
   - Configure load balancer to distribute traffic

2. **Process**:
   - Start with a small percentage (e.g., 5%) of traffic to Octane
   - Monitor performance and errors
   - Gradually increase traffic to Octane
   - Eventually decommission PHP-FPM servers

3. **Benefits**:
   - Reduced risk by limiting exposure
   - Real-world validation with actual users
   - Ability to compare performance directly

### Feature Flag Deployment

Use feature flags to control the migration:

1. **Setup**:
   - Implement a feature flag system
   - Configure your application to use either PHP-FPM or Octane based on the flag

2. **Process**:
   - Deploy the dual-capable application
   - Enable Octane for internal users first
   - Gradually enable for more user segments
   - Monitor and adjust as needed

3. **Benefits**:
   - Fine-grained control over the migration
   - Ability to target specific user segments
   - Easy rollback by toggling the flag

## Monitoring and Optimization

### Key Metrics to Monitor

After migration, monitor these key metrics:

1. **Performance Metrics**:
   - Request throughput
   - Response time (average, p95, p99)
   - CPU and memory usage
   - Database connection count

2. **Stability Metrics**:
   - Error rates
   - Worker restarts
   - Memory growth over time
   - Request timeouts

3. **Business Metrics**:
   - User experience metrics
   - Conversion rates
   - Page load times
   - API response times

### Optimization Opportunities

Once stable, consider these optimizations:

1. **Worker Tuning**:
   - Adjust worker count based on CPU cores and workload
   - Configure worker lifecycle (max requests, TTL)
   - Optimize memory limits

2. **Connection Pooling**:
   - Fine-tune database connection pools
   - Implement connection reuse for external services
   - Consider persistent Redis connections

3. **Preloading**:
   - Use the `warm` configuration to pre-resolve frequently used services
   - Implement PHP's OPcache preloading for core classes
   - Preload commonly used data into shared memory

4. **Advanced Features**:
   - Implement WebSockets for real-time features
   - Use coroutines for concurrent operations (Swoole)
   - Leverage task workers for CPU-intensive operations

## Conclusion

Migrating to Laravel Octane is a significant step that can dramatically improve your application's performance. By following a structured approach and addressing common challenges proactively, you can ensure a smooth transition with minimal disruption.

Remember that each application is unique, and you may encounter specific challenges not covered in this guide. Always test thoroughly in a staging environment that mirrors production as closely as possible before deploying to production.

The effort invested in a careful migration will pay dividends in improved performance, reduced infrastructure costs, and enhanced user experience. 