---
category: note-2021
published: true
layout: post
title: 开发适用于 Laravel 的 Composer 包
description: 及时记录，定期回顾
---

# 开发适用于 Laravel 的 Composer 包

## 目录结构

可以通过 `composer init` 创建初始化目录



```bash
project
	|_ src
	|_ tests
	|_ vendor
	|_ .editorconfig
	|_ .gitattributes
	|_ .gitignore
	|_ .php_cs
	|_ composer.json
	|_ composer.lock
	|_ phpunit.xml.dist
	|_ README.md
```



主要代码放在 `project/src` 下面



```bash
project
	|_ src
		|_ config
			|_ example.php
		|_ database
			|_ migrations
				|_ create_example_tables.php
		|_ Example.php
		|_ ServiceProvider.php
```





## 为 Laravel 集成优化

1. 创建 `src/ServiceProvider.php`

   

```php
<?php

namespace Catname\Example;

class ServiceProvider extends \Illuminate\Support\ServiceProvider
{
    // Laravel 扩展包延迟注册
    protected $defer = true;

    public function register()
    {
        $this->app->singleton(Example::class, function(){
            return new Example();
        });

        $this->app->alias(Example::class, 'example');
    }

    // Laravel 扩展包延迟注册
    public function provides()
    {
        return [Example::class, 'example'];
    }
    
    public function boot()
    {
        // 发布配置文件
        $this->publishes([
            __DIR__.'/config/example.php' => config_path('example.php'),
        ], 'config');
        $this->publishes([
            __DIR__.'/database/migrations/create_example_tables.php' => $this->getMigrationFileName('create_example_tables.php'),
        ], 'migrations');
    }
}
```



发布文件的命令( 此命令会发布所有文件，配置文件和数据库迁移文件)



```bash
php artisan vendor:publish --provider="Catname\Example\ServiceProvider"
```



通过定义的标签发布指定的文件



```bash
php artisan vendor:publish --tag=config
```



[更多发布资源有关内容参考](https://learnku.com/docs/laravel/8.5/packages/10394#d2bcef)



2. 在 `composer.json` 中添加下列内容

   

```json
.
.
.
"extra": {
    "laravel": {
        "providers": [
            "Catname\\Example\\ServiceProvider"
        ]
    }
}
```



这样便无需手动注册服务提供器。









