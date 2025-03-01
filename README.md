# Laravel Octane Server Options: Comprehensive Comparison

This repository provides an in-depth analysis of the different server options available for Laravel Octane: FrankenPHP, Swoole, and RoadRunner.

## Overview

Laravel Octane supercharges your application's performance by serving your application using high-powered application servers. This repository helps you choose the right server option for your specific use case.

## Quick Comparison

| Feature | FrankenPHP | Swoole | RoadRunner |
|---------|------------|--------|------------|
| Performance | Very Good | Excellent | Good |
| Ease of Setup | Excellent | Good | Very Good |
| PHP Extension Required | No | Yes | No |
| Memory Usage | Good | Excellent | Good |
| Docker Integration | Excellent | Good | Good |
| WebSocket Support | Limited | Excellent | Good |
| Community Support | Growing | Extensive | Good |
| Compatibility | Very Good | Good | Excellent |

## Detailed Documentation

For a comprehensive analysis of each server option, please refer to the following documentation:

- [FrankenPHP In-Depth Guide](docs/frankenphp.md)
- [Swoole In-Depth Guide](docs/swoole.md)
- [RoadRunner In-Depth Guide](docs/roadrunner.md)
- [Performance Benchmarks](docs/benchmarks.md)
- [Decision Framework](docs/decision-framework.md)
- [Migration Strategies](docs/migration-strategies.md)
- [Common Issues & Solutions](docs/troubleshooting.md)

## Decision Framework

### Choose FrankenPHP if:
- You're deploying with Docker/containers
- You want the simplest "all-in-one" solution
- You need built-in HTTPS without configuration
- You're starting a new project without legacy constraints

### Choose Swoole if:
- Absolute maximum performance is critical
- You need WebSocket capabilities
- You're comfortable managing PHP extensions
- You want access to coroutines and advanced features
- You're building a high-traffic application

### Choose RoadRunner if:
- You can't install PHP extensions
- You need maximum PHP library compatibility
- You're modernizing an existing application
- You want a balance of performance and simplicity
- Your team is more comfortable with traditional PHP patterns

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is open-sourced software licensed under the MIT license. 