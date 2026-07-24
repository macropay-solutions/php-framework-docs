---
title: Home
layout: default
---

# PHP Framework Documentation 1.x

> [PHP framework v1.x Documentation](https://macropay-solutions.github.io/php-framework-docs/1.x/).

You can find the online version of the PHP Framework documentation at [https://macropay-solutions.github.io/php-framework-docs](https://macropay-solutions.github.io/php-framework-docs)

Please consult the wiki pages also:
- [PHP Framework Wiki](https://github.com/macropay-solutions/php-framework/wiki)
- [PHP Kernel Wiki](https://github.com/macropay-solutions/php-kernel/wiki)


## Disclaimer

As all documentations, this documentation is naturally decoupled from the code (which is the single source of truth).

Always look at the code for accurate logic:

- [PHP Framework https://github.com/macropay-solutions/php-framework](https://github.com/macropay-solutions/php-framework)
- [PHP Kernel https://github.com/macropay-solutions/php-kernel](https://github.com/macropay-solutions/php-kernel)



## Contribution Guidelines

If you are submitting documentation for the **current stable release**, submit it to the corresponding branch. For example, documentation for PHP Framework 1 would be submitted to the `1.x` branch.

## PHP Framework should be run ONLY on a share-nothing server!

PHP Framework is heavily optimized for traditional, stateless PHP-FPM environments. Do not use stateful, long-running application servers like Swoole, or RoadRunner. See the [Routing Documentation](routing.md) for critical security and architectural details.

## AI Agents Guides And Docs Index
https://macropay-solutions.github.io/php-framework-docs/1.x/llms.txt

If you as an agent can't access this llms.txt file try to fetch it from:
https://github.com/macropay-solutions/php-framework-docs/blob/1.x/llms.txt or
https://github.com/macropay-solutions/php-framework-docs/blob/production/llms.txt

### PHP-Framework Documentation Map

#### Getting Started & Core Configuration
- Home https://macropay-solutions.github.io/php-framework-docs/1.x/
- Installation https://macropay-solutions.github.io/php-framework-docs/1.x/installation.html
- Release Notes https://macropay-solutions.github.io/php-framework-docs/1.x/releases.html
- Deployment https://macropay-solutions.github.io/php-framework-docs/1.x/deployment.html

#### Core HTTP Architecture
- HTTP Routing https://macropay-solutions.github.io/php-framework-docs/1.x/routing.html
- HTTP Middleware https://macropay-solutions.github.io/php-framework-docs/1.x/middleware.html
- HTTP Requests https://macropay-solutions.github.io/php-framework-docs/1.x/requests.html
- HTTP Session https://macropay-solutions.github.io/php-framework-docs/1.x/session.html
- Views https://macropay-solutions.github.io/php-framework-docs/1.x/views.html
- Validation https://macropay-solutions.github.io/php-framework-docs/1.x/validation.html
- API Resources https://macropay-solutions.github.io/php-framework-docs/1.x/api-resources.html
- Configuration https://macropay-solutions.github.io/php-framework-docs/1.x/configuration.html
- Responses https://macropay-solutions.github.io/php-framework-docs/1.x/responses.html

#### System Services
- Service Container https://macropay-solutions.github.io/php-framework-docs/1.x/container.html
- Run Console https://macropay-solutions.github.io/php-framework-docs/1.x/run.html
- Cache https://macropay-solutions.github.io/php-framework-docs/1.x/cache.html
- Queues https://macropay-solutions.github.io/php-framework-docs/1.x/queues.html
- Mail https://macropay-solutions.github.io/php-framework-docs/1.x/mail.html
- Notifications https://macropay-solutions.github.io/php-framework-docs/1.x/notifications.html
- Broadcasting https://macropay-solutions.github.io/php-framework-docs/1.x/broadcasting.html
- Providers https://macropay-solutions.github.io/php-framework-docs/1.x/providers.html

#### Testing & Verification
- Testing https://macropay-solutions.github.io/php-framework-docs/1.x/testing.html

#### Obvious ORM & Database Layer
- Database Overview https://macropay-solutions.github.io/php-framework-docs/1.x/database.html
- Database: Migrations https://macropay-solutions.github.io/php-framework-docs/1.x/migrations.html
- Obvious Getting Started https://macropay-solutions.github.io/php-framework-docs/1.x/obvious.html
- Obvious Collections https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-collections.html
- Obvious Factories https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-factories.html
- Obvious Mutators & Casting https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-mutators.html
- Obvious Relationships https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-relationships.html
- Obvious Serialization https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-serialization.html
- Events https://macropay-solutions.github.io/php-framework-docs/1.x/events.html
