---
category: note-2022
published: true
layout: post
title: Laravel 开发踩坑日常记录
description: 及时记录，定期回顾
---



### 一般部署



```bash
1. git clone git@xxx.xxx.com:xxxxx/xxx.git && cd xxx
2. composer install
3. cp env.example env.
4. php artisan key:generate
5. php artisan jwt:secret
6. # 填写好 .env 文件中的各项配置，特别是数据库，Redis 等

   # 开发环境运行数据库迁移文件 并且运行填充文件 填充测试数据
   # 注意！运行此命令前，请先备份 Laravel-admin 管理后台数据，具体查看本文档-其他杂项
7. php artisan migrate:refresh --seed
   # 正式环境不需要填充所有数据
   php artisan migrate
```



### 开发规范

在 [Laravel 项目开发规范](https://learnku.com/docs/laravel-specification/7.x/whats-the-use-of-standards/7588) 的基础上，按下列规范进行开发。  

* 数据库**必须**使用 `数据库迁移文件` 做统一版本管理，方便多人协作开发和后期切换数据库需要。
* 所有金额相关的字段，字段类型为 `int` 单位统一为 `分`。
* 所有时间操作统一使用 `Carbon` 类。
* 模型关联后的关联查询**必须**使用 `预加载 with()` 防止 N+1 问题，除非情况不允许。
* 获取当前运行环境 `app('env')` 。
* 添加自定义配置文件，放在 `app/config/` ，使用 `config('文件名.键名')` 获取相对应的配置。
* 表单验证类 统一继承 基类 `BaseRequest` ，因为基类中重写了表单验证未通过后返回的消息结构。
* 写日志用 Monolog ，可在 `config/logging.php` 中自定义 channel 实现不同文件记录不同日志。
* 新建时间类字段使用 datetime 类型，防止出现 `Y2K38` 问题。



### 初始化

#### 修改 `config/app.php` 配置





```php
<?php

return [

    ...

    /*
    |--------------------------------------------------------------------------
    | Application Timezone
    |--------------------------------------------------------------------------
    |
    | Here you may specify the default timezone for your application, which
    | will be used by the PHP date and date-time functions. We have gone
    | ahead and set this to a sensible default for you out of the box.
    |
    */

    'timezone' => 'Asia/Shanghai',

    /*
    |--------------------------------------------------------------------------
    | Application Locale Configuration
    |--------------------------------------------------------------------------
    |
    | The application locale determines the default locale that will be used
    | by the translation service provider. You are free to set this value
    | to any of the locales which will be supported by the application.
    |
    */

    'locale' => 'zh_CN',

    /*
    |--------------------------------------------------------------------------
    | Application Fallback Locale
    |--------------------------------------------------------------------------
    |
    | The fallback locale determines the locale to use when the current one
    | is not available. You may change the value to correspond to any of
    | the language folders that are provided through your application.
    |
    */

    'fallback_locale' => 'en',

    /*
    |--------------------------------------------------------------------------
    | Faker Locale
    |--------------------------------------------------------------------------
    |
    | This locale will be used by the Faker PHP library when generating fake
    | data for your database seeds. For example, this will be used to get
    | localized telephone numbers, street address information and more.
    |
    */

    'faker_locale' => 'zh_CN',

    ...

];
```



#### 添加自定义辅助函数文件 `app/helpers.php`

在 `composer.json` 中添加下面内容



```json
...
"autoload": {
    "psr-4": {
        ...
    },
    "files": [
        "app/helpers.php"
    ]
},
...
```



添加后，运行以下命令重新加载文件



```bash
composer dump-autoload
```





#### 路由分组

`app/Providers/RouteServiceProvider.php`



```php
...
public function boot()
{
    $this->configureRateLimiting();

    $this->routes(function () {
        Route::prefix('api')
            ->middleware('api')
            ->namespace($this->namespace)
            ->group(base_path('routes/api.php'));
        Route::prefix('api')
            ->middleware('api')
            ->namespace($this->namespace)
            ->group(base_path('routes/common.php'));

        Route::middleware('web')
            ->namespace($this->namespace)
            ->group(base_path('routes/web.php'));
    });
}
...
```





### 请求

#### 规范返回格式，给所有 API 请求加上请求头

1. 新建中间件 `AcceptHeader.php`

   

   ```php
   ...
   public function handle(Request $request, Closure $next)
   {
       $request->headers->set('Accept', 'application/json');
       return $next($request);
   }
   ...
   ```

   

2. 注册中间件 `app/Http/Kernel.php`

   
   
   ```php
   ...
   protected $middlewareGroups = [
       'web' => [
              ...
       ],
   
       'api' => [
           \App\Http\Middleware\AcceptHeader::class,
           ...
       ],
   ];
   ...
   ```
   
   

### 路由

#### 模型存在，却抛出 `No query results for model [App\\Models\\xxxx]` 异常

这个通常由路由绑定出的问题，注意有绑定模型的路由，同路径的路由需要放在没绑定路由的后面

例如：/posts/create 和 /posts/{post} 的是同路径，/posts/{post} 必须放在 /posts/create 后面

一定注意路由的先后顺序



### 管道 Pipeline



```php
<?php

public function create(Request $request)
{
    $pipes = [
        RemoveBadWords::class,
        ReplaceLinkTags::class,
        RemoveScriptTags::class
    ];
    $post = app(Pipeline::class)
        ->send($request->content)
        ->through($pipes)
        ->then(function ($content) {
            return Post::create(['content' => 'content']);
        });
    // return any type of response
}
```



### 中间件

#### 获取当前请求的 URL



```php
// 不带主机域名和参数
$request->path();          //结果： api/home/test

// 不带主机域名，带 query 参数
$request->getRequestUrl(); //结果： /api/home/test?id=1

// 带主机域名，不带 query 参数
$request->url();           //结果： http://xxx.test/api/home/test
// 注意最前面 '/' 的区别
```



#### 获取当前请求的 HTTP Method



```php
$request->method(); //结果： POST
```





### 捕获异常

#### 捕获确切的数据库错误

如果您想捕获 Eloquent Query 异常，请使用特定的 `QueryException` 代替默认的 Exception 类，您将能够获得错误的确切 SQL 代码。



```php
try {
    // 一些 Eloquent/SQL 声明
} catch (\Illuminate\Database\QueryException $e) {
    if ($e->getCode() === '23000') { // integrity constraint violation
        return back()->withError('Invalid data');
    }
}
```



### 控制器

#### 模型依赖注入



```php
<?php
namespace App\Http\Controllers;

class ExampleController extends Controller
{
    public function show(Model $model)
	{
        ...
    	// 如果需要关联查询
        $model->load('relation');
        ...
	}
    ...
}
```





### 服务提供者

#### 创建



```bash
php artisan make:provider NotificationServiceProvider
```



#### 注册 `config/app.php`



```php
<?php

return [
    ...
    'providers' => [
        ...
        App\Providers\NotificationServiceProvider::class,
        ...
    ],
    ...
];
```



#### 用例：监听日志，指定等级日志推送消息



```php
<?php

namespace App\Providers;

use App\Jobs\noticeIssue;
use Illuminate\Support\ServiceProvider;

class NotificationServiceProvider extends ServiceProvider
{
    ...
    public function boot()
    {
        \Log::listen(function ($e) {
            // 这里监听到日志后，判断日志的等级，然后做相应处理
            $noticeMsg = [
                'level'   => $e->level, // 日志的等级
                'message' => $e->message,
                'context' => $e->context,
            ];
            $data = [
                // 推送数据
            ];
            noticeIssue::dispatch($data)->onQueue('issues');
        });
    }
}

```





### 数据库操作

#### 更新 JSON 字段

更新 JSON 字段时，你可以使用 `->` 语法访问 JSON 对象中相应的值。注意，此操作只能支持 MySQL 5.7+ 和 PostgreSQL 9.5+ ：



```php
$affected = DB::table('users')
              ->where('id', 1)
              ->update(['options->enabled' => true]);
```



### Artisan 命令

#### 命令使用参数 [详情参考](https://learnku.com/docs/laravel/8.5/artisan/10381#arguments)



```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Cache;

class Test extends Command
{
    protected $signature = 'example:test {--test}';
    protected $description = '参数示范';
    ...
    public function handle()
    {
        // 获取选项的值
        $option = $this->option('test');
        // $this->argument('argument') 获取参数的值
        return 0;
    }
}
```



### 集合

#### 举例



```php
	protected function shouldPassThrough($request)
    {
        // 下面这些路由不验证权限
        $excepts = array_merge(config('admin.auth.excepts', []), [
            'authorizations', // 登录
            'authorizations/current', // 刷新 Token // 注销
        ]);
        return collect($excepts)
            ->map('admin_base_path')// 加上前缀
            ->contains(function ($except) use ($request) {
                // 去掉最前面的 '/'
                if ($except !== '/') {
                    $except = trim($except, '/');
                }
                
                return $request->is($except);
            });
    }
```



##### map



##### contains



### 队列

#### 手动失败



```php
$this->fail('队列失败。');
```



### ORM 模型

#### 根据子表的值筛选主表的项 Model->with()->whereHas() *



```php
$permissions = User::select('id', 'name')
    			->with('roles.permissions:id,name,title')
    			->whereHas('roles', function (Builder $query) {
            		$query->where('model_id',auth()->id())
                          ->select('id','name','title');
        		})->get();
```



主表在 只有字表中 model_id 为指定值时才会被查出来。

#### 嵌套预加载

在一个 Eloquent 语句中预加载所有书籍作者及其联系方式：



```php
$books = Book::with('author.contacts')->get();
```



book 的关联 author 的关联 contacts

#### 多对多关联

给当前项目添加用户 id 为 1 的中间关联数据，并且附加 user_type 字段



```php
$project->allusers()->attach(1, ['user_type' => 3]);
```



删除用户 id 为 1 的中间关联数据，注意，当前项目下所有 id 为 1 的都会被删除，**不适用于一个项目下有多个不同 user_type 但相同用户 id 的情况**



```php
$project->allusers()->detach(1);
```





### 用户授权策略类 (Policy)

必须在同名控制器中才会生效？？

#### 注意神坑！！如果在用户授权策略类中查询过关联模型，那么关联模型会被加入结果集中，需要在返回前调用 `$model->unsetRelation('relationName')` 去掉。

### [模型观察者 (Observer)](https://learnku.com/docs/laravel/8.5/eloquent/10409#cab083)

#### 使用

生成观察者类  



```bash
php artisan make:observer UserObserver --model=User
```



在 `AppServiceProvider` 服务提供者中注册观察者



```php
use App\Models\User;
use App\Observers\UserObserver;

/**
 * Register any events for your application.
 *
 * @return void
 */
public function boot()
{
    User::observe(UserObserver::class);
}
```



#### 调用顺序

modelA  
ObserverA

监听 modelA 的 created 事件



```php
$A = MODELA::create($data);

next
```



实际运行顺序  
create->observer->next

#### 踩过的坑

##### 坑1

###### 问题场景：工单系统微信登录返回预期外数据，前台用户登录一切正常，后台用户登录会多返回后台用户的角色信息，但是登录控制器里面没有任何和 roles 有关的代码

前台普通用户登录后返回：



```json
{
          "code": 200,
          "messages": "登录成功",
          "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczpcL1wvaXNzdWVjcy5xaWZ1eGlvbmcuY29tXC9hcGlcL3dlY2hhdFwvYXV0aG9yaXphdGlvbnMiLCJpYXQiOjE2Mzk5OTI2NTgsImV4cCI6MTYzOTk5OTg1OCwibmJmIjoxNjM5OTkyNjU4LCJqdGkiOiJoc1NLMFptRGRrZGl0RDlRIiwic3ViIjoxNCwicHJ2IjoiMjNiZDVjODk0OWY2MDBhZGIzOWU3MDFjNDAwODcyZGI3YTU5NzZmNyIsInJvbGUiOiJ1c2VyIn0.GEUahFc4XlxaFvdV4MclSJ4CvLCCdOKKJGQwvZ04I5s",
          "token_type": "Bearer",
          "expires": 1639999858,
          "is_binding_mobile": true,
          "is_staff": true,
          "userInfo": {
              "id": 14,
              "nick_name": "昵称",
              "real_name": "王老五",
              "mobile": "18888888888",
              "account_status": 1,
              "avatar_url": "https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epk6XJfVGqsW5y4zFjzWy3SvYN1KRHQYLTKGnbUanOfBeW4sR0z6ZyNBgtCVmOceicTlnyBcaTPbWA/132",
              "gender": 0,
              "company": "成都市企服熊科技有限公司",
              "job": "PHP后端",
              "created_at": "2021-12-20 16:26:37",
              "updated_at": "2021-12-20 17:30:58"
          }
      }
```



后台用户微信登录后返回



```json
{
          "code": 200,
          "messages": "登录成功",
          "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczpcL1wvaXNzdWVjcy5xaWZ1eGlvbmcuY29tXC9hcGlcL3dlY2hhdFwvYXV0aG9yaXphdGlvbnMiLCJpYXQiOjE2Mzk5OTI2NTgsImV4cCI6MTYzOTk5OTg1OCwibmJmIjoxNjM5OTkyNjU4LCJqdGkiOiJoc1NLMFptRGRrZGl0RDlRIiwic3ViIjoxNCwicHJ2IjoiMjNiZDVjODk0OWY2MDBhZGIzOWU3MDFjNDAwODcyZGI3YTU5NzZmNyIsInJvbGUiOiJ1c2VyIn0.GEUahFc4XlxaFvdV4MclSJ4CvLCCdOKKJGQwvZ04I5s",
          "token_type": "Bearer",
          "expires": 1639999858,
          "is_binding_mobile": true,
          "is_staff": true,
          "userInfo": {
              "id": 14,
              "nick_name": "昵称",
              "real_name": "王老五",
              "mobile": "18888888888",
              "account_status": 1,
              "avatar_url": "https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epk6XJfVGqsW5y4zFjzWy3SvYN1KRHQYLTKGnbUanOfBeW4sR0z6ZyNBgtCVmOceicTlnyBcaTPbWA/132",
              "gender": 0,
              "company": "成都市企服熊科技有限公司",
              "job": "PHP后端",
              "created_at": "2021-12-20 16:26:37",
              "updated_at": "2021-12-20 17:30:58",
              "roles": [
                  {
                      "id": 1,
                      "name": "basicUser",
                      "title": "基础用户",
                      "guard_name": "api",
                      "created_at": "2021-12-17 16:36:08",
                      "updated_at": "2021-12-17 16:36:08",
                      "pivot": {
                          "model_id": 14,
                          "role_id": 1,
                          "model_type": "App\\Models\\User"
                      }
                  }
              ]
          }
      }
```



###### 产生原因

后台有一个基础用户权限，在添加和更新用户的时候会通过用户模型监听器监听到事件，然后添加权限，产生这个问题的原因在 更新事件，监听到更新事件后，会检查用户是否拥有基础权限，如果没有则加上，在检查之后，会将 roles 加入到 user 的结果集中。

###### 解决办法

1. 在模型监听器更新事件中，处理完后，执行 `$model->unsetRelation('roles');`

2. 在不需要返回 roles 的地方使用 `$model->unsetRelation('roles');`

推荐使用方法 1







### Composer

#### 线上服务器更新指定 Composer 包

1. 在本地运行

   

   ```bash
   composer update xxx/xxxxxxxx
   ```



2. 将更新后的  `composer.lock` 文件提交到服务器

3. 服务器终端运行

   

   ```bash
   composer install
   ```

   

   

### 钉钉

#### 单聊机器人

按文档传参数会提示：发送可交互卡片的时候提示 非场景群；

解决方法：机器人应该用   appKey ：机器人appKey

#### 回调事件加解密

##### 事件订阅注册

在注册时不要按照文档说的用企业 corpId，而是用自建应用的 appKey，否则会报 AES 解密错误

注册代码



```php
<?php
	$crypt = new DingCallbackCrypto();
    $text = $crypt->getDecryptMsg($request->msg_signature, $request->timestamp, $request->nonce, $request->encrypt);
    \Log::channel('hotlog')->debug($text);
    $res = $crypt->getEncryptedMap("success");
	//return response($res);
    $data = json_decode($res);
    return $data;
```



### 其他杂项，技巧

* 本地开发调接口的时候，需要带上 JWT-Token ,可以通过自定义 Artisan 命令（`php artisan` 命令可以看到当前所有的 artisan 命令）生成有效期一年的JWT-Token，将 Token 存入开发环境变量中（ Postman 等）

* 获取自己（服务端）的 IP 地址：`\request()->server('SERVER_ADDR');`

* 本地开发自定义网址的时候不能以 .dev 结尾，否则 Chrome 会强制跳转 https 报证书错误。

* 在运行 `php artisan migrate:refresh --seed` 之前，注意备份 laravel-admin 后台数据 `php artisan admin:export-seed`

* 注意在需要登录状态的控制器的构造方法中加载 api 中间件（如果路由中已经加载 auth:api 中间件，则不需要在控制器中单独加载），这样授权策略类等，才能通过 JWT-Token 获取到当前登录的用户 `$this->middleware('auth:api');`

* 返回用户信息相关，统一使用 `UserResource` 对敏感信息进行了脱敏，如果因为业务需要，可以这样显示敏感信息 `return (new UserResource($user))->additional(['code' => 200, 'messages' => 'success'])->showSensitiveFields();`

* ！！注意访问器这个天坑！！强烈建议所有 status 相关的判断都用 `getRawOriginal()` 获取原始值，避免因为有访问器导致判断错误，业务出现 bug 。在 Laravel 7 以后，如果不想使用访问器返回的数据，可以用 `$model->getRawOriginal('字段名')` 获取原始数据。(Laravel 7 之前的版本是 `getOriginal()` )，通过关联模型查询也可以使用 `$model->relationships->getRawOriginal('字段名')`

* 注意 Laravel-Admin 可能会占用 名为 admin 的 guard ，如果要新增自定义 guard ，避免使用 admin 这个名字。

* 注意在返回模型前，如果对模型进行过关联查询，关联查询的数据会被添加到模型中，如果后面不需要这些数据，记得在关联查询完成后，直接调用 unsetRelation()。

* 通过 `Carbon::now()->getPreciseTimestamp()` 可以获取指定精度的时间戳。
  
  
  
  ```php
  /*
  * @example getPreciseTimestamp()   1532087464437474 (微秒最大精度)
  * @example getPreciseTimestamp(6)  1532087464437474
  * @example getPreciseTimestamp(5)  153208746443747  (1/100000 秒精度)
  * @example getPreciseTimestamp(4)  15320874644375   (1/10000 秒精度)
  * @example getPreciseTimestamp(3)  1532087464437    (毫秒 precision)
  * @example getPreciseTimestamp(2)  153208746444     (1/100 秒精度)
  * @example getPreciseTimestamp(1)  15320874644      (1/10 秒精度)
  * @example getPreciseTimestamp(0)  1532087464       (秒精度)
  * @example getPreciseTimestamp(-1) 153208746        (10 秒精度)
  * @example getPreciseTimestamp(-2) 15320875         (100 秒精度)
  */
  ```
  
  
  
* 如果需要同样内容在表中只有一条数据，可以用 `firstOrCreate()`

  

  ```php
  // 按名称检索航班或在它不存在时创建它...
  $flight = Flight::firstOrCreate([
      'name' => 'London to Paris'
  ]);
  ```

  

* 特别注意：线上环境，用宝塔面板初始化的时候，如果执行 airtisan 命令出现错误，日志文件会用 root 的身份创建，导致运行在 www 的代码没有权限进行读写。





### 依赖注入

> 当A类需要依赖于B类，也就是说需要在A类中实例化B类的对象来使用时候，如果B类中的功能发生改变，也会导致A类中使用B类的地方也要跟着修改，导致A类与B类高耦合。

依赖注入本质是在外部将 B 类实例化后，再传到 A 类，这样对 B 类的修改扩展，不用改 A 的代码。

> 不需要自己内容修改，改成由外部传递。这种由外部负责其依赖需求的行为，我们可以称其为 “控制反转（IoC）”。
>
> 那什么是依赖注入呢？，其实上面的例子也算是依赖注入，不是由自己内部 new 对象或者实例，通过构造函数，或者方法传入的都属于 依赖注入（DI） 。

[更多](https://learnku.com/docs/laravel-core-concept/5.5/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5,%E6%8E%A7%E5%88%B6%E7%BF%BB%E8%BD%AC,%E5%8F%8D%E5%B0%84/3017)

### 服务容器

TODO



### Laravel-Permission

#### 给角色同步权限

前端用 Json 传入需要同步的全部 Id，通过 whereIn() 过滤掉 Id 不存在的权限



```php
public function syncPermissions(Role $role, Request $request)// TODO 表单验证
{
    $permissionIds = json_decode($request->permissionIds, true);
    try {
        $permissions = Permission::whereIn('id', $permissionIds)->get();
        // 将多个权限同步到一个角色
        $role->syncPermissions($permissions);
    } catch (\Exception $e) {
        \Log::channel('errors')->notice('[角色权限]给角色('.$role->name.')添加权限('.$request->permissionIds.')时出错，错误提示：'.$e->getMessage());
        return response([
            'code' => 500,
            'messages' => '添加失败',
            'data' => []
        ]);
    }

    return response([
        'code' => 200,
        'messages' => '角色添加权限成功',
        'data' => []
    ]);
}
```



### 爬虫相关

#### HTTP/2

##### 特征



```http
:authority: pc.api.longyiyy.com 
:method: GET
:path: /api/goods/list?page=1&category_id=&sort=&order=&is_activity=0&is_youx=&is_xy=&no_image=0
:scheme: https
accept: application/json, text/plain, */*
```



HTTP/2 最明显的特征就是从浏览器 Header 头里面可以看到 `:`开头的伪 Header 头，用 PHP Guzzle 请求的时候，不需要在 Header 头里面加入伪 Header 头，只需要在请求选项中，设定使用 HTTP/2 协议。



```php
$response = Http::withHeaders([
			'accept' => 'application/json, text/plain, */*',
            'accept-encoding' => 'gzip, deflate, br',
            'accept-language' => 'zh-CN,zh;q=0.9',
            'cookie' => '...',
            'origin' => '...',
            'sec-ch-ua' => '...',
            'sec-ch-ua-mobile' => '...',
            'sec-ch-ua-platform' => '...',
            'sec-fetch-dest' => '...',
            'sec-fetch-mode' => '...',
            'sec-fetch-site' => '...',
            'user-agent' => '...'
		])->withOptions([
            'version' => 2.0,
		])->get($url, [...])
```





### Excel

#### 导出 Excel

##### [Laravel Excel](https://docs.laravel-excel.com/3.1/getting-started/)

安装扩展包



```bash
composer require maatwebsite/excel
```



创建输出类

```bash
php artisan make:export ExampleExport --model=Example
```



`app/Exports/ExampleExport.php`



```php
<?php

namespace App\Exports;

use App\Models\Example;
use Maatwebsite\Excel\Concerns\FromCollection;
use Maatwebsite\Excel\Concerns\WithColumnWidths;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;

class ProductsExport implements FromCollection, WithColumnWidths, WithHeadings, WithMapping
{
    public function collection()
    {
        ...
        // 获取数据
        ...
        return $collections;
    }

    // 指定列的宽度，单位不是像素
    public function columnWidths(): array
    {
        return [
            'A' => 10,
            'B' => 10,
            ...
        ];
    }

    // 指定每一列对应的数据源
    public function map($row): array
    {
        return [
            $row->id,
            '',
            $row->licenseNumber,
            ...
        ];
    }

    // 标题
    public function headings(): array
    {
        return [
            '商品编号',
            '商品条码',
            ...
        ];
    }
}

```



导出到磁盘



```php
Excel::store(new ExampleExport(), 'examples_'.now()->timestamp.'.xlsx');
```



控制器中导出下载



```php
return Excel::download(new ExampleExport, 'examples.xlsx');
```



[更多导出详情](https://docs.laravel-excel.com/3.1/exports/)

