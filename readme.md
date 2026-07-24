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
- Installation https://macropay-solutions.github.io/php-framework-docs/1.x/installation
- Release Notes https://macropay-solutions.github.io/php-framework-docs/1.x/releases
- Deployment https://macropay-solutions.github.io/php-framework-docs/1.x/deployment

#### Core HTTP Architecture
- HTTP Routing https://macropay-solutions.github.io/php-framework-docs/1.x/routing
- HTTP Middleware https://macropay-solutions.github.io/php-framework-docs/1.x/middleware
- HTTP Requests https://macropay-solutions.github.io/php-framework-docs/1.x/requests
- HTTP Session https://macropay-solutions.github.io/php-framework-docs/1.x/session
- Views https://macropay-solutions.github.io/php-framework-docs/1.x/views
- Validation https://macropay-solutions.github.io/php-framework-docs/1.x/validation
- API Resources https://macropay-solutions.github.io/php-framework-docs/1.x/api-resources
- Configuration https://macropay-solutions.github.io/php-framework-docs/1.x/configuration
- Responses https://macropay-solutions.github.io/php-framework-docs/1.x/responses
- Controllers https://macropay-solutions.github.io/php-framework-docs/1.x/controllers
- Encryption https://macropay-solutions.github.io/php-framework-docs/1.x/encryption
- Authentication https://macropay-solutions.github.io/php-framework-docs/1.x/authentication
- Authorization https://macropay-solutions.github.io/php-framework-docs/1.x/authorization

#### System Services
- Service Container https://macropay-solutions.github.io/php-framework-docs/1.x/container
- Run Console https://macropay-solutions.github.io/php-framework-docs/1.x/run
- Cache https://macropay-solutions.github.io/php-framework-docs/1.x/cache
- Queues https://macropay-solutions.github.io/php-framework-docs/1.x/queues
- Mail https://macropay-solutions.github.io/php-framework-docs/1.x/mail
- Notifications https://macropay-solutions.github.io/php-framework-docs/1.x/notifications
- Broadcasting https://macropay-solutions.github.io/php-framework-docs/1.x/broadcasting
- Providers https://macropay-solutions.github.io/php-framework-docs/1.x/providers

#### Testing & Verification
- Testing https://macropay-solutions.github.io/php-framework-docs/1.x/testing

#### Obvious ORM & Database Layer
- Database Overview https://macropay-solutions.github.io/php-framework-docs/1.x/database
- Database: Migrations https://macropay-solutions.github.io/php-framework-docs/1.x/migrations
- Obvious Getting Started https://macropay-solutions.github.io/php-framework-docs/1.x/obvious
- Obvious Collections https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-collections
- Obvious Factories https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-factories
- Obvious Mutators & Casting https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-mutators
- Obvious Relationships https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-relationships
- Obvious Serialization https://macropay-solutions.github.io/php-framework-docs/1.x/obvious-serialization
- Events https://macropay-solutions.github.io/php-framework-docs/1.x/events
