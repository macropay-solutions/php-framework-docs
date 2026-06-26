---
title: Release Notes
description: Release notes, versioning scheme, and support policy for PHP Framework.
context: releases
---

# Release Notes

- [Versioning Scheme](#versioning-scheme)
- [Support Policy](#support-policy)
- [PHP-Framework 1](#php-framework-1)

<a name="versioning-scheme"></a>
## Versioning Scheme

PHP-Framework follows [Semantic Versioning](https://semver.org) and release schedule. Major releases occur every 2 years (~Q4), while minor and patch releases may be released as often as every week. Minor and patch releases should **never** contain breaking changes.

When referencing the PHP-Framework or its components from your application or package, you should always use a version constraint such as `^1.0`, since major releases of PHP-Framework do include breaking changes. However, we strive to always ensure you may update to a new major release in one day or less.

<a name="named-arguments"></a>
#### Named Arguments

[Named arguments](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) are not covered by PHP-Framework's backwards compatibility guidelines. We may choose to rename function arguments when necessary in order to improve the PHP-Framework codebase. Therefore, using named arguments when calling PHP-Framework methods should be done cautiously and with the understanding that the parameter names may change in the future.

<a name="support-policy"></a>
## Support Policy

For PHP-Framework releases, bug fixes and security fixes are provided for 3 years, in strict alignment with the PHP-Kernel support lifecycle. In addition, please review the database versions [supported by PHP-Framework](/database#introduction).

<div class="overflow-auto" markdown="1">

| Version | PHP (*)              | Release | Bug Fixes Until | Security Fixes Until |
|---------|----------------------|---------|-----------------|----------------------|
| 1       | 8.2 - 8.4 (8.5 Beta) | Q4 2026 | 30 Nov 2029     | 30 Nov 2029          |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>End of life</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>Security fixes only</div>
    </div>
</div>

(*) Supported PHP versions

<a name="php-framework-1"></a>
## PHP-Framework 1

PHP-Framework 1 represents the ultimate distillation of the PHP-Kernel architecture into a pure, lightning-fast API backend. In this release, the kernel has been heavily decoupled to achieve sub-millisecond boot times and a baseline memory footprint of just 0.37 MB.

By strictly aligning with modern PHP standards and purging all monolithic dependencies, PHP-Framework 1 delivers our fastest, most memory-efficient API experience to date.

<a name="high-speed-request-hydration"></a>
### High-Speed Request Hydration

To achieve unprecedented execution speeds, PHP-Framework 1 implements a custom Request capture engine that completely bypasses legacy `SymfonyRequest::createFromGlobals()` instantiation.

By directly reading early stream parsing for `application/json` payloads from `php://input` and leveraging native support for PHP 8.4's `\request_parse_body()`, the framework achieves Symfony 7.4 parity with maximum velocity. Furthermore, redundant file scrubbing has been streamlined into a strictly typed static phase (`Request::cleanFiles()`), entirely eliminating boot-cycle overhead.

<a name="next-gen-service-container"></a>
### Next-Generation Service Container

The underlying Service Container has received a massive, ground-up rewrite. PHP-Framework 1 introduces **Two-Tier DI Caching** with encapsulated cache loading and deep OPcache protection.

By shifting legacy properties to a flat-array structure and fusing the JSON request bag directly into the primary InputBag, the container drastically reduces its baseline memory footprint. New utility methods like `Container::isInInstances()` power fast-path runtime identity verification, making the container's relay logic practically stateless and incredibly fast.

<a name="pipeline-short-circuiting"></a>
### Pipeline Short-Circuiting & Lifecycle

The Request lifecycle and HTTP Kernel have been deeply optimized to save critical execution cycles. PHP-Framework 1 introduces **Pipeline Short-Circuiting**: if the middleware stack is empty or disabled, the application entirely skips the instantiation of the `Pipeline` component, dropping the request straight into the target controller action to eliminate allocation overhead.

In addition, the dispatcher loop now utilizes identity-gated rebinding (`!==`), preventing redundant request re-bindings unless a middleware explicitly replaces the request pointer. The framework also enforces a strict `MacropaySolutions\Framework\Http\Request` parameter signature, guaranteeing type safety from boot to termination.

<a name="modern-ecosystem-upgrades"></a>
### Modern Ecosystem Upgrades

PHP-Framework 1 fully embraces the cutting edge of the PHP ecosystem, requiring **PHP 8.2 or higher**.

The framework has been completely upgraded to utilize the **Symfony 7.4** component suite, bringing strict PHP return types (`: int`) to Console Commands and internal architecture. Additionally, date and time manipulation is now powered by **Carbon 3**, ensuring your application benefits from the latest advancements in immutable date handling.
