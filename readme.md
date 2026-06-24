---
title: Home
layout: default
---

# PHP Rramework Documentation 1.x

> [PHP framework v1.x Documentation](https://macropay-solutions.github.io/maravel-docs/10.x/).

You can find the online version of the PHP Framework documentation at [https://macropay-solutions.github.io/maravel-docs](https://macropay-solutions.github.io/php-framework-docs)

Please consult the wiki pages also:
- [PHP Framework Wiki](https://github.com/macropay-solutions/php-framework/wiki)
- [PHP Kernel Wiki](https://github.com/macropay-solutions/php-kernel/wiki)


## Disclaimer

As all documentations, this documentation is naturally decoupled from the code (which is the single source of truth).

Always look at the code for accurate logic:

- [PHP Framework](https://github.com/macropay-solutions/php-framework)
- [PHP Kernel](https://github.com/macropay-solutions/php-kernel)



## Contribution Guidelines

If you are submitting documentation for the **current stable release**, submit it to the corresponding branch. For example, documentation for PHP Framework 1 would be submitted to the `1.x` branch.

## PHP Framework should be run ONLY on a share-nothing server!

PHP Framework is heavily optimized for traditional, stateless PHP-FPM environments. Do not use stateful, long-running application servers like Swoole, or RoadRunner. See the [Routing Documentation](routing.md) for critical security and architectural details.
