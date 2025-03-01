# Common Issues & Solutions

This document provides solutions to common issues you might encounter when using Laravel Octane with different server options: FrankenPHP, Swoole, and RoadRunner.

## General Octane Issues

### Memory Leaks

**Symptoms:**
- Gradually increasing memory usage over time
- Workers being killed due to exceeding memory limits
- Degraded performance after the application runs for a while

**Solutions:**
1. **Identify the source:**
   ```bash
   # Add memory reporting to your application
   $memoryUsage = round(memory_get_usage() / 1024 / 1024, 2);
   Log::info("Memory usage: {$memoryUsage}MB");
   ```

2. **Reset static properties:**
   ```php
   // In AppServiceProvider
   public function register()
   {
       $this->app->terminating(function () {
           // Reset static properties in your classes
           YourClass::resetStaticProperties();
       });
   }
   ```

3. **Configure worker recycling:**
   ```php
   // config/octane.php
   return [
       'max_requests' => 1000, // Recycle workers after 1000 requests
   ];
   ```

### Database Connection Issues

**Symptoms:**
- "Too many connections" errors
- Stale or lost connections after periods of inactivity
- Connection timeout errors

**Solutions:**
1. **Reconnect after each request:**
   ```php
   // In AppServiceProvider
   public function register()
   {
       $this->app->terminating(function ($app) {
           $app->make('db')->reconnect();
       });
   }
   ```

2. **Configure connection pooling:**
   ```php
   // config/database.php
   'mysql' => [
       // ...
       'pool' => [
           'enabled' => true,
           'min' => 5,
           'max' => 20,
       ],
   ],
   ```

3. **Implement connection heartbeats:**
   ```php
   // In a scheduled command
   DB::select('SELECT 1');
   ```

### Session State Issues

**Symptoms:**
- Unexpected session behavior
- Sessions being shared between requests
- Authentication state bleeding between requests

**Solutions:**
1. **Reset session manager:**
   ```php
   // In AppServiceProvider
   public function register()
   {
       $this->app->terminating(function ($app) {
           $app->make('session')->driver()->reset();
       });
   }
   ```

2. **Use appropriate session driver:**
   ```php
   // .env
   SESSION_DRIVER=redis
   ```

3. **Avoid storing request-specific data in static properties:**
   ```php
   // Bad practice
   class UserContext
   {
       public static $currentUser;
   }
   
   // Better approach
   class UserContext
   {
       private $currentUser;
       
       public function setCurrentUser($user)
       {
           $this->currentUser = $user;
       }
       
       public function getCurrentUser()
       {
           return $this->currentUser;
       }
   }
   ```

## FrankenPHP Specific Issues

### HTTPS Certificate Issues

**Symptoms:**
- Certificate errors in browser
- Self-signed certificate warnings
- Certificate not renewing automatically

**Solutions:**
1. **Ensure proper domain configuration:**
   ```
   # Caddyfile
   yourdomain.com {
       # Configuration here
   }
   ```

2. **Check DNS configuration:**
   ```bash
   # Verify DNS resolution
   dig yourdomain.com
   ```

3. **Manual certificate renewal:**
   ```bash
   docker exec -it your-frankenphp-container caddy reload
   ```

### Static File Serving Issues

**Symptoms:**
- 404 errors for static files
- Slow loading of assets
- Incorrect MIME types

**Solutions:**
1. **Configure proper static file handling:**
   ```
   # Caddyfile
   yourdomain.com {
       root * /app/public
       php_server {
           resolve_root_symlink
       }
       file_server
   }
   ```

2. **Add proper MIME types:**
   ```
   # Caddyfile
   yourdomain.com {
       # ...
       mime .woff application/font-woff
       mime .woff2 application/font-woff2
   }
   ```

### Docker Integration Issues

**Symptoms:**
- Container fails to start
- Permission issues with files
- Volume mounting problems

**Solutions:**
1. **Fix permissions:**
   ```dockerfile
   # Dockerfile
   FROM dunglas/frankenphp
   
   # ...
   
   RUN chown -R www-data:www-data /app
   USER www-data
   ```

2. **Correct volume mounting:**
   ```yaml
   # docker-compose.yml
   services:
     app:
       image: dunglas/frankenphp
       volumes:
         - .:/app:cached
   ```

3. **Use health checks:**
   ```yaml
   # docker-compose.yml
   services:
     app:
       # ...
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost/health"]
         interval: 30s
         timeout: 10s
         retries: 3
   ```

## Swoole Specific Issues

### Extension Installation Problems

**Symptoms:**
- "Extension not found" errors
- Compilation errors during installation
- Incompatible extension version

**Solutions:**
1. **Install correct version:**
   ```bash
   pecl install swoole-4.8.12
   ```

2. **Check PHP compatibility:**
   ```bash
   php -i | grep PHP
   # Ensure Swoole supports your PHP version
   ```

3. **Manual compilation:**
   ```bash
   git clone https://github.com/swoole/swoole-src.git
   cd swoole-src
   phpize
   ./configure
   make && make install
   ```

### Coroutine Deadlocks

**Symptoms:**
- Requests hanging indefinitely
- Timeouts in coroutine operations
- Deadlocks between coroutines

**Solutions:**
1. **Set timeout for coroutines:**
   ```php
   Swoole\Coroutine::set([
       'socket_timeout' => 5,
       'max_coroutine' => 3000,
   ]);
   ```

2. **Avoid blocking operations in coroutines:**
   ```php
   // Avoid this in coroutines
   sleep(10);
   
   // Use this instead
   Swoole\Coroutine::sleep(10);
   ```

3. **Use channels for communication:**
   ```php
   $channel = new Swoole\Coroutine\Channel(1);
   
   go(function () use ($channel) {
       $result = doSomething();
       $channel->push($result);
   });
   
   $result = $channel->pop(5); // Wait for result with 5s timeout
   ```

### WebSocket Connection Issues

**Symptoms:**
- WebSocket connections dropping
- Failed WebSocket handshakes
- Connection limit issues

**Solutions:**
1. **Configure proper limits:**
   ```php
   // config/octane.php
   return [
       'swoole' => [
           'options' => [
               'max_conn' => 10000,
               'heartbeat_check_interval' => 60,
               'heartbeat_idle_time' => 600,
           ],
       ],
   ];
   ```

2. **Implement ping/pong:**
   ```javascript
   // Client-side
   const ws = new WebSocket('wss://example.com/ws');
   
   setInterval(() => {
       if (ws.readyState === WebSocket.OPEN) {
           ws.send(JSON.stringify({type: 'ping'}));
       }
   }, 30000);
   ```

3. **Handle disconnections gracefully:**
   ```php
   Octane::webSocketHandler(function ($request, $websocket) {
       $websocket->on('disconnect', function ($websocket) {
           // Clean up resources
           Redis::srem('online_users', Auth::id());
       });
       
       return $websocket;
   });
   ```

## RoadRunner Specific Issues

### Binary Installation Problems

**Symptoms:**
- "Command not found" errors
- Incompatible binary version
- Permission issues with the binary

**Solutions:**
1. **Manual installation with specific version:**
   ```bash
   wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.12.1/roadrunner-2.12.1-linux-amd64.tar.gz
   tar -xzf roadrunner-2.12.1-linux-amd64.tar.gz
   chmod +x ./rr
   ```

2. **Use the installation script with version:**
   ```bash
   curl -s https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/install.sh | bash -s -- -v 2.12.1
   ```

3. **Fix permissions:**
   ```bash
   chmod +x ./rr
   sudo mv ./rr /usr/local/bin/
   ```

### Worker Process Management

**Symptoms:**
- Workers not starting
- Workers crashing unexpectedly
- Too many or too few workers

**Solutions:**
1. **Configure appropriate worker settings:**
   ```yaml
   # .rr.yaml
   http:
     address: 0.0.0.0:8080
     pool:
       num_workers: 8
       max_jobs: 1000
       supervisor:
         exec_ttl: 60s
         max_worker_memory: 128
   ```

2. **Implement graceful shutdown:**
   ```yaml
   # .rr.yaml
   server:
     command: "php worker.php"
     relay: "pipes"
     env:
       - OCTANE_GRACEFUL_SHUTDOWN=true
   ```

3. **Debug worker issues:**
   ```bash
   ./rr serve -d -v
   ```

### Request Handling Issues

**Symptoms:**
- Requests timing out
- "Worker not ready" errors
- PSR-7 compatibility issues

**Solutions:**
1. **Increase timeout settings:**
   ```yaml
   # .rr.yaml
   http:
     address: 0.0.0.0:8080
     max_request_size: 32MB
     read_timeout: 60s
     write_timeout: 60s
   ```

2. **Fix PSR-7 compatibility:**
   ```php
   // Make sure you're using compatible PSR-7 implementation
   composer require nyholm/psr7
   ```

3. **Handle large file uploads:**
   ```yaml
   # .rr.yaml
   http:
     address: 0.0.0.0:8080
     max_request_size: 128MB
     uploads:
       forbid: [".php", ".exe", ".bat"]
       dir: "/tmp"
   ```

## Performance Tuning

### Slow Response Times

**Symptoms:**
- Higher than expected response times
- Performance degradation under load
- Inconsistent response times

**Solutions:**
1. **Optimize worker count:**
   ```php
   // For Swoole/FrankenPHP, generally:
   // CPU cores * 2 = optimal worker count
   
   // config/octane.php
   return [
       'workers' => 8, // Adjust based on your CPU cores
   ];
   ```

2. **Enable OPcache:**
   ```ini
   ; php.ini
   opcache.enable=1
   opcache.validate_timestamps=0
   opcache.memory_consumption=128
   opcache.interned_strings_buffer=16
   ```

3. **Implement caching:**
   ```php
   // Use Redis for caching
   Cache::remember('key', $seconds, function () {
       return expensiveOperation();
   });
   ```

### High Memory Usage

**Symptoms:**
- Workers using more memory than expected
- OOM (Out of Memory) errors
- Frequent worker recycling

**Solutions:**
1. **Limit memory per worker:**
   ```php
   // config/octane.php
   return [
       'swoole' => [
           'options' => [
               'max_request' => 500,
               'worker_max_memory' => 512, // MB
           ],
       ],
   ];
   ```

2. **Optimize database queries:**
   ```php
   // Use chunking for large datasets
   User::query()->chunk(1000, function ($users) {
       foreach ($users as $user) {
           // Process user
       }
   });
   ```

3. **Profile memory usage:**
   ```php
   // Add to middleware or specific routes
   $initialMemory = memory_get_usage();
   
   // After request processing
   $finalMemory = memory_get_usage();
   Log::info("Memory used: " . ($finalMemory - $initialMemory) / 1024 / 1024 . "MB");
   ```

### CPU Bottlenecks

**Symptoms:**
- High CPU usage
- Slow response under load
- Workers maxing out CPU

**Solutions:**
1. **Offload CPU-intensive tasks:**
   ```php
   // For Swoole
   Octane::task(function () {
       return processCpuIntensiveTask();
   });
   
   // For other servers, use queue
   dispatch(function () {
       processCpuIntensiveTask();
   });
   ```

2. **Implement caching for expensive computations:**
   ```php
   $result = Cache::remember('expensive-computation', 3600, function () {
       return performExpensiveComputation();
   });
   ```

3. **Use appropriate task worker count:**
   ```php
   // config/octane.php
   return [
       'swoole' => [
           'options' => [
               'worker_num' => 8,
               'task_worker_num' => 4, // For CPU-bound tasks
           ],
       ],
   ];
   ```

## Debugging Techniques

### Request Debugging

**Techniques:**
1. **Enable detailed logging:**
   ```php
   // config/logging.php
   'channels' => [
       'octane' => [
           'driver' => 'daily',
           'path' => storage_path('logs/octane.log'),
           'level' => 'debug',
       ],
   ],
   ```

2. **Add request/response logging middleware:**
   ```php
   class LogRequestsMiddleware
   {
       public function handle($request, Closure $next)
       {
           $start = microtime(true);
           $response = $next($request);
           $duration = microtime(true) - $start;
           
           Log::channel('octane')->info("Request: {$request->fullUrl()} - Duration: {$duration}s");
           
           return $response;
       }
   }
   ```

3. **Use dump server for debugging:**
   ```bash
   # Install
   composer require --dev symfony/var-dumper
   
   # Run dump server
   php artisan dump-server
   
   # In your code
   dump($variable); // Output goes to dump server
   ```

### Worker Debugging

**Techniques:**
1. **Monitor worker status:**
   ```bash
   # For Swoole
   watch -n 1 "ps aux | grep swoole"
   
   # For RoadRunner
   ./rr workers -i 1
   ```

2. **Implement worker ID logging:**
   ```php
   // In AppServiceProvider
   public function boot()
   {
       $this->app->terminating(function () {
           $workerId = getmypid();
           Log::info("Request processed by worker: {$workerId}");
       });
   }
   ```

3. **Debug worker lifecycle:**
   ```php
   // config/octane.php
   return [
       'listeners' => [
           WorkerStarting::class => [
               function () {
                   Log::info('Worker starting: ' . getmypid());
               },
           ],
           WorkerStopping::class => [
               function () {
                   Log::info('Worker stopping: ' . getmypid());
               },
           ],
       ],
   ];
   ```

### Performance Profiling

**Techniques:**
1. **Use Blackfire.io:**
   ```bash
   # Install Blackfire agent
   curl -s https://packagecloud.io/install/repositories/blackfire/agent/script.deb.sh | sudo bash
   sudo apt-get install blackfire
   
   # Profile with Blackfire
   blackfire curl http://your-app.test/endpoint
   ```

2. **Implement custom profiling:**
   ```php
   // In a middleware
   $startTime = microtime(true);
   $startMemory = memory_get_usage();
   
   $response = $next($request);
   
   $endTime = microtime(true);
   $endMemory = memory_get_usage();
   
   Log::info("Time: " . ($endTime - $startTime) . "s, Memory: " . ($endMemory - $startMemory) / 1024 / 1024 . "MB");
   
   return $response;
   ```

3. **Use Laravel Telescope in development:**
   ```bash
   # Install Telescope
   composer require laravel/telescope --dev
   
   # Publish assets
   php artisan telescope:install
   php artisan migrate
   ```

## Deployment Troubleshooting

### Failed Deployments

**Symptoms:**
- Application not starting after deployment
- 502 Bad Gateway errors
- Supervisor failing to start workers

**Solutions:**
1. **Check logs:**
   ```bash
   # Supervisor logs
   sudo tail -f /var/log/supervisor/supervisord.log
   
   # Application logs
   tail -f storage/logs/laravel.log
   ```

2. **Verify permissions:**
   ```bash
   # Fix storage permissions
   sudo chown -R www-data:www-data storage bootstrap/cache
   sudo chmod -R 775 storage bootstrap/cache
   ```

3. **Test configuration:**
   ```bash
   # For RoadRunner
   ./rr validate -c .rr.yaml
   
   # For Swoole
   php -r "if (extension_loaded('swoole')) echo 'Swoole installed';"
   ```

### Load Balancer Issues

**Symptoms:**
- Inconsistent routing
- WebSocket connection failures
- Session persistence problems

**Solutions:**
1. **Configure sticky sessions:**
   ```
   # Nginx load balancer
   upstream backend {
       server backend1.example.com;
       server backend2.example.com;
       sticky cookie srv_id expires=1h domain=.example.com path=/;
   }
   ```

2. **Set appropriate timeouts:**
   ```
   # Nginx configuration
   server {
       # ...
       proxy_connect_timeout 300s;
       proxy_send_timeout 300s;
       proxy_read_timeout 300s;
       # ...
   }
   ```

3. **Configure WebSocket proxying:**
   ```
   # Nginx configuration
   location /ws {
       proxy_pass http://backend;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_set_header Host $host;
   }
   ```

### Scaling Issues

**Symptoms:**
- Performance degradation when scaling up
- Resource contention
- Inconsistent behavior across instances

**Solutions:**
1. **Use centralized session storage:**
   ```php
   // config/session.php
   return [
       'driver' => 'redis',
       'connection' => 'session',
   ];
   ```

2. **Implement proper cache invalidation:**
   ```php
   // Use cache tags for easier invalidation
   Cache::tags(['users', 'user:'.$user->id])->put('user:'.$user->id, $user, 3600);
   
   // Invalidate specific tags
   Cache::tags(['user:'.$user->id])->flush();
   ```

3. **Configure proper queue system:**
   ```php
   // config/queue.php
   return [
       'default' => 'redis',
       'connections' => [
           'redis' => [
               'driver' => 'redis',
               'connection' => 'default',
               'queue' => env('REDIS_QUEUE', 'default'),
               'retry_after' => 90,
               'block_for' => null,
           ],
       ],
   ];
   ```

## Conclusion

This troubleshooting guide covers common issues you might encounter when working with Laravel Octane and its different server options. Remember that each application is unique, and you may need to adapt these solutions to your specific environment and requirements.

When troubleshooting, always start by checking the logs and isolating the problem to a specific component. Methodical debugging and a good understanding of how Octane and your chosen server option work will help you resolve issues efficiently.

If you encounter persistent issues, consider reaching out to the community through:
- [Laravel Octane GitHub Issues](https://github.com/laravel/octane/issues)
- [Laravel Discord](https://discord.gg/laravel)
- Server-specific communities (Swoole, RoadRunner, or FrankenPHP) 