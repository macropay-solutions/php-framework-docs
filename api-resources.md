---
title: API Resources
description: Guide to API presentation mapping and why JsonResource is not recommended in PHP Framework.
context: api-resources
---

# API Resources

<a name="why-we-dropped-jsonresource"></a>
## Why We Don't Recommend JsonResource

Standard frameworks rely on `JsonResource` classes to format data. If an API endpoint returns 10,000 rows, the framework instantiates 10,000 ephemeral wrapper objects in memory. Under concurrent load, this floods the PHP heap, causes massive garbage collection latency spikes, and frequently triggers `memory_limit` crashes.

To make PHP-Kernel's APIs **Bulletproof**, we no longer recommend using the `JsonResource`. PHP-Crufd-Wizard Lib Suite processes flat, scalar arrays directly through a stateless middleware pipeline, allowing your server to format and stream all rows with a flat memory footprint.

<a name="documentation-and-usage"></a>
## Documentation & Usage

In the PHP-Kernel ecosystem, presentation mapping is handled via **Middleware Decorators** provided by the CRUFD Wizard suite.

Instead of duplicating the documentation here, please refer directly to the official package repositories for comprehensive guides on schema masking, route application, virtual columns, and zero-allocation streaming:

- **[PHP CRUFD Wizard - Official Documentation](https://github.com/macropay-solutions/php-crufd-wizard)**
- **[PHP CRUFD Wizard Decorator - Official Documentation](https://github.com/macropay-solutions/php-crufd-wizard-decorator)**

You can generate all the needed classes using:
- **[PHP CRUFD Wizard Generator - Official Documentation](https://github.com/macropay-solutions/php-crufd-wizard-generator)**
