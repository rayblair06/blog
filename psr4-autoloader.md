---
title: "PSR4 - Autoloader"
author: "Ray Blair"
date_published: 24-03-2024
---

## PSR4 - Autoloader

Autoloading is a commonly used mechanism that automatically loads classes and/or files into memory when they are needed, without having to explicitly include them within each file. In the context of PHP, Composer uses this technique instead of typically using `require` or `include` statements to manually load each class before the class is used. Composer can generate a list of mappings defined in `composer.json` and autoload third party dependencies without much requirement from the user.

Outside of Composer, it is possible to write your own autoloader function so that PHP automatically calls classes that haven't been loaded yet. The `spl_autoload_register()` function allows you to register one or more autoloaders.

Some of the benefits of autoloading are:
- Simplicity
- Efficiency
- Maintainability

Although writing your own autoloader can be fun and is a great learning experience, it is always better to use tried and tested methods such as Composer's autoloader when building applications that may go into production.

Resources:
https://www.php-fig.org/psr/psr-4/
https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader-examples.md

![PSR 4 - Autoloader](images/psr4-autoloader.jpeg)

