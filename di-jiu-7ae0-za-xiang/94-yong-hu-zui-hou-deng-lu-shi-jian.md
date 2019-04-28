## 最近活跃时间

接下来我们将在用户的个人中心头像下显示此用户的『最后活跃时间』。

在 Web 应用中，很多时候应用的效率瓶颈会出现在『数据库系统』中，所以作为一个合格的 Web 开发工程师，我们要严格控制好我们的数据库开销。之前我们已经对一些不需要频繁修改的数据做了缓存，如『活跃用户』数据和『资源推荐』数据，以此减少数据库读取的压力。然而，在数据库中操作中，『写入』对数据库造成的压力，要远比『读取』压力高得多。

想要准确地跟踪用户的最后活跃时间，就必须在用户每一次请求服务器时都做记录，我们使用的主数据是 MySQL，也就是说每当用户访问一个页面，我们都将 MySQL 数据库里的`users`表写入数据。当我们有很多用户频繁访问站点时，这将会是数据库的一笔巨大开销。

我们可以使用 Redis 来记录用户的访问时间，Redis 运行在机器的内存上，读写效率都极快。不过为了保证数据的完整性，我们需要定期将 Redis 数据同步到数据库中，否则一旦 Redis 出问题或者执行了 Redis 清理操作，用户的『最后活跃时间』将会丢失。

基本思路如下：

1. 记录 - 通过中间件过滤用户所有请求，记录用户访问时间到 Redis 按日期区分的哈希表；
2. 同步 - 新建命令，计划任务每天运行一次此命令，将昨日哈希表里的数据同步到数据库中，并删除；
3. 读取 - 优先读取当日哈希表里 Redis 里的数据，无数据则使用数据库中的值。

## 1. 记录时间

我们将利用[Laravel 中间件](https://learnku.com/docs/laravel/5.7/middleware)功能来实现用户访问时间的记录。

Laravel 中间件提供了一种方便的机制来过滤进入应用的 HTTP 请求。例如，Laravel 内置了一个中间件来验证用户的身份认证。如果用户没有通过身份认证，中间件会将用户重定向到登录界面。但是，如果用户被认证，中间件将允许该请求进一步进入该应用。这就是我们在控制器的构造方法中使用`auth`中间件：

```
public function __construct()
{
    $this->middleware('auth', ['except' => ['index', 'show']]);
}
```

当然，除了身份认证以外，中间件还可以用来执行各种任务。例如：CORS 中间件可以负责为所有离开应用的响应添加合适的头部信息；日志中间件可以记录所有传入应用的请求。Laravel 自带了一些中间件，包括身份验证、CSRF 保护等。所有这些中间件都位于`app/Http/Middleware`目录中。

Laravel 的中间件从执行时机上分『前置中间件』和『后置中间件』，前置中间件是应用初始化完成以后立刻执行，此时控制器路由还未分配、控制器还未执行、视图还未渲染。后置中间件是即将离开应用的响应，此时控制器已将渲染好的视图返回，我们可以在后置中间件里修改响应。两者的区别在于书写方式的不同：

前置中间件：

```
<?php

namespace App\Http\Middleware;

use Closure;

class BeforeMiddleware
{
    public function handle($request, Closure $next)
    {
        // 这是前置中间件，在还未进入 $next 之前调用

        return $next($request);
    }
}
```

后置中间件：

```
<?php

namespace App\Http\Middleware;

use Closure;

class AfterMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        // 这是后置中间件，$next 已经执行完毕并返回响应 $response，
        // 我们可以在此处对响应进行修改。

        return $response;
    }
}
```

> 注意他们的区别在于`$next($request)`的执行位置，而非类的命名或者其他。

我们将选择『前置中间件』的方式来实现用户登录时间的记录。

### 第一步。创建中间件

运行以下命令，生成中间件类文件：

```
$ php artisan make:middleware RecordLastActivedTime
```

### 第二步。注册中间件

想让中间件在应用的每个 HTTP 请求期间运行，我们还需要在`app/Http/Kernel.php`类中对中间件进行注册。

为了方便大家理解，我将`Kernel.php`里的代码做了注释，同时在 Web 中间件组中注册了 RecordLastActivedTime 中间件，这样每一次 Web 请求都会运行我们的 RecordLastActivedTime 类里的`handle()`方法：

_app/Http/Kernel.php_

    <?php

    namespace App\Http;

    use Illuminate\Foundation\Http\Kernel as HttpKernel;

    class Kernel extends HttpKernel
    {
        // 全局中间件
        protected $middleware = [
            // 检测是否应用是否进入『维护模式』
            // 见：https://learnku.com/docs/laravel/5.7/configuration#maintenance-mode
            \App\Http\Middleware\CheckForMaintenanceMode::class,

            // 检测表单请求的数据是否过大
            \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,

            // 对提交的请求参数进行 PHP 函数 `trim()` 处理
            \App\Http\Middleware\TrimStrings::class,

            // 将提交请求参数中空子串转换为 null
            \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,

            // 修正代理服务器后的服务器参数
            \App\Http\Middleware\TrustProxies::class,
        ];

        /**
         * The application's route middleware groups.
         *
         * @var array
         */
        protected $middlewareGroups = [

            // Web 中间件组，应用于 routes/web.php 路由文件，
            // 在 RouteServiceProvider 中设定
            'web' => [
                // Cookie 加密解密
                \App\Http\Middleware\EncryptCookies::class,

                // 将 Cookie 添加到响应中
                \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,

                // 开启会话
                \Illuminate\Session\Middleware\StartSession::class,

                // \Illuminate\Session\Middleware\AuthenticateSession::class,

                // 将系统的错误数据注入到视图变量 $errors 中
                \Illuminate\View\Middleware\ShareErrorsFromSession::class,

                // 检验 CSRF ，防止跨站请求伪造的安全威胁
                // 见：https://learnku.com/docs/laravel/5.7/csrf
                \App\Http\Middleware\VerifyCsrfToken::class,

                // 处理路由绑定
                // 见：https://learnku.com/docs/laravel/5.7/routing#route-model-binding
                \Illuminate\Routing\Middleware\SubstituteBindings::class,

                // 强制用户邮箱认证
                \App\Http\Middleware\EnsureEmailIsVerified::class,

                // 记录用户最后活跃时间
                \App\Http\Middleware\RecordLastActivedTime::class,
            ],

            // API 中间件组，应用于 routes/api.php 路由文件，
            // 在 RouteServiceProvider 中设定
            'api' => [
                // 使用别名来调用中间件
                // 请见：https://learnku.com/docs/laravel/5.7/middleware#为路由分配中间件
                'throttle:60,1',
                'bindings',
            ],
        ];

        // 中间件别名设置，允许你使用别名调用中间件，例如上面的 api 中间件组调用
        protected $routeMiddleware = [

            // 只有登录用户才能访问，我们在控制器的构造方法中大量使用
            'auth' => \Illuminate\Auth\Middleware\Authenticate::class,

            // HTTP Basic Auth 认证
            'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,

            // 处理路由绑定
            // 见：https://learnku.com/docs/laravel/5.7/routing#route-model-binding
            'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class,

            'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,

            // 用户授权功能
            'can' => \Illuminate\Auth\Middleware\Authorize::class,

            // 只有游客才能访问，在 register 和 login 请求中使用，只有未登录用户才能访问这些页面
            'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,

            // 签名认证，在找回密码章节里我们讲过
            'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,

            // 访问节流，类似于 『1 分钟只能请求 10 次』的需求，一般在 API 中使用
            'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,

            // Laravel 自带的强制用户邮箱认证的中间件，为了更加贴近我们的逻辑，已被重写
            'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
        ];

        // 设定中间件优先级，此数组定义了除『全局中间件』以外的中间件执行顺序
        // 可以看到 StartSession 永远是最开始执行的，因为 StartSession 后，
        // 我们才能在程序中使用 Auth 等用户认证的功能。
        protected $middlewarePriority = [
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\Authenticate::class,
            \Illuminate\Session\Middleware\AuthenticateSession::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            \Illuminate\Auth\Middleware\Authorize::class,
        ];
    }

### 第三步。书写中间件类

_app/Http/Middleware/RecordLastActivedTime.php_

```
<?php

namespace App\Http\Middleware;

use Closure;
use Auth;

class RecordLastActivedTime
{
    public function handle($request, Closure $next)
    {
        // 如果是登录用户的话
        if (Auth::check()) {
            // 记录最后登录时间
            Auth::user()->recordLastActivedAt();
        }

        return $next($request);
    }
}
```

我们将业务逻辑封装与 User 类中，事实上，我们要将记录和读取的相关逻辑放置于单独的 Trait 中：

_app/Models/Traits/LastActivedAtHelper.php_

```
<?php

namespace App\Models\Traits;

use Redis;
use Carbon\Carbon;

trait LastActivedAtHelper
{
    // 缓存相关
    protected $hash_prefix = 'larabbs_last_actived_at_';
    protected $field_prefix = 'user_';

    public function recordLastActivedAt()
    {
        // 获取今天的日期
        $date = Carbon::now()->toDateString();

        // Redis 哈希表的命名，如：larabbs_last_actived_at_2017-10-21
        $hash = $this->hash_prefix . $date;

        // 字段名称，如：user_1
        $field = $this->field_prefix . $this->id;

        // 当前时间，如：2017-10-21 08:35:15
        $now = Carbon::now()->toDateTimeString();

        // 数据写入 Redis ，字段已存在会被更新
        Redis::hSet($hash, $field, $now);
    }
}
```

然后在 User 模型引用：

_app/Models/User.php_

```
<?php
.
.
.
class User extends Authenticatable implements MustVerifyEmailContract
{
    use Traits\LastActivedAtHelper;

    .
    .
    .
}
```

### 第四步。测试一下

如果我们的代码可用的话，当我们处于登录状态时，每一次访问网站都会将当前的时间，记录在 Redis 的哈希表里。

接下来我们登录 1，2，3 三个不同用户访问网页，以此来制造一些数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/2C47fyvNBX.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/2C47fyvNBX.gif!large)

接下来修改`recordLastActivedAt`方法，在得到哈希表名称后，使用`hGetAll`获取哈希表里的全部数据，并使用 Laravel 自带的调试方法`dd()`打印出来：

```
  public function recordLastActivedAt()
    {
        // 获取今天的日期
        $date = Carbon::now()->toDateString();

        // Redis 哈希表的命名，如：larabbs_last_actived_at_2017-10-21
        $hash = $this->hash_prefix . $date;

        // 字段名称，如：user_1
        $field = $this->field_prefix . $this->id;

        dd(Redis::hGetAll($hash));

        // 当前时间，如：2017-10-21 08:35:15
        $now = Carbon::now()->toDateTimeString();

        // 数据写入 Redis ，字段已存在会被更新
        Redis::hSet($hash, $field, $now);
    }
```

刷新页面即可看到，我们已经能正常记录用户的访问时间：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/gBHYi6aT7M.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/gBHYi6aT7M.png!large)

测试完成后删除`dd(Redis::hGetAll($hash));`调用。

## 2. 同步到数据库

接下来把记录下来的时间同步到数据库中。我们将设置一个 24 小时运行一次的计划任务，此任务负责将 Redis 中用户最后登录时间同步到数据库中。

### 第一步。数据库添加字段

首先我们需要在`users`表中新建字段，用以存储

```
$ php artisan make:migration add_last_actived_at_to_users_table --table=users
```

代码迁移内容：

_database/migrations/{timestamp}\_add\_last\_actived\_at\_to\_users\_table.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddLastActivedAtToUsersTable extends Migration
{
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->timestamp('last_actived_at')->nullable();
        });
    }

    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('last_actived_at');
        });
    }
}
```

生效到数据库中：

```
$ php artisan migrate
```

### 第二步。新建 Artisan 命令

我们需要新建命令，以供计划任务调用：

```
$ php artisan make:command SyncUserActivedAt --command=larabbs:sync-user-actived-at
```

编辑命令类：

_app/Console/Commands/SyncUserActivedAt.php_

```
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\User;

class SyncUserActivedAt extends Command
{
    protected $signature = 'larabbs:sync-user-actived-at';
    protected $description = '将用户最后登录时间从 Redis 同步到数据库中';

    public function handle(User $user)
    {
        $user->syncUserActivedAt();
        $this->info("同步成功！");
    }
}
```

为方便代码管理，我们将具体的业务代码放置于 LastActivedAtHelper Trait 中：

_app/Models/Traits/LastActivedAtHelper.php_



    <?php
    .
    .
    .

    trait LastActivedAtHelper
    {
        .
        .
        .

        public function syncUserActivedAt()
        {
            // 获取昨天的日期，格式如：2017-10-21
            $yesterday_date = Carbon::yesterday()->toDateString();

            // Redis 哈希表的命名，如：larabbs_last_actived_at_2017-10-21
            $hash = $this->hash_prefix . $yesterday_date;

            // 从 Redis 中获取所有哈希表里的数据
            $dates = Redis::hGetAll($hash);

            // 遍历，并同步到数据库中
            foreach ($dates as $user_id => $actived_at) {
                // 会将 `user_1` 转换为 1
                $user_id = str_replace($this->field_prefix, '', $user_id);

                // 只有当用户存在时才更新到数据库中
                if ($user = $this->find($user_id)) {
                    $user->last_actived_at = $actived_at;
                    $user->save();
                }
            }

            // 以数据库为中心的存储，既已同步，即可删除
            Redis::del($hash);
        }
    }

### 第三步。测试 Artisan 命令

接下来我们需测试下代码，已确保 Redis 中的数据能正常同步到数据库中。在开始之前，我们获取的是昨日的数据，不方便测试，我们需将以下这一行代码：

```
$yesterday_date = Carbon::yesterday()->toDateString();
```

临时修改为今天的日期，这样就能将我们的上一个测试制造的数据获取到：

```
$yesterday_date = Carbon::now()->toDateString();
```

修改完成后，命令行执行：

```
$ php artisan larabbs:sync-user-actived-at
```

执行成功后打开数据库，查看用户的`last_actived_at`字段是否有数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/l3RLGRPUlr.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/l3RLGRPUlr.png!large)

测试完成后请将`$yesterday_date`变量赋值代码还原。

### 第四步。任务调度

我们还需要在每天零时`00:00`自动对数据进行同步。Laravel 任务调度可以轻松帮我们实现此功能。

在前面开发活跃用户章节中我们已经做了 Cron 设置，此处我们只需在`Kernel.php`的`schedule()`方法中新增任务调度即可：

_app/Console/Kernel.php_

```
<?php
.
.
.
class Kernel extends ConsoleKernel
{
    .
    .
    .
    protected function schedule(Schedule $schedule)
    {
        // $schedule->command('inspire')
        //          ->hourly();
        // 每隔一个小时执行一遍
        $schedule->command('larabbs:calculate-active-user')->hourly();
        // 每日零时执行一次
        $schedule->command('larabbs:sync-user-actived-at')->dailyAt('00:00');
    }
    .
    .
    .
}
```

## 3. 数据读取

接下来我们要将记录下来的数据在用户的个人空间里显示出来。

### 第一步。配置访问器

我们将使用 Eloquent 的[访问器](https://learnku.com/docs/laravel/5.7/eloquent-mutators#defining-an-accessor)来实现此功能。当我们从实例中获取某些属性值的时候，访问器允许我们对 Eloquent 属性值进行动态修改。

访问器的命名规范与修改器类似，只是将`set`换成`get`而已：

_app/Models/Traits/LastActivedAtHelper.php_

```
<?php
.
.
.

trait LastActivedAtHelper
{
    .
    .
    .

    public function getLastActivedAtAttribute($value)
    {
        // 获取今天的日期
        $date = Carbon::now()->toDateString();

        // Redis 哈希表的命名，如：larabbs_last_actived_at_2017-10-21
        $hash = $this->hash_prefix . $date;

        // 字段名称，如：user_1
        $field = $this->field_prefix . $this->id;

        // 三元运算符，优先选择 Redis 的数据，否则使用数据库中
        $datetime = Redis::hGet($hash, $field) ? : $value;

        // 如果存在的话，返回时间对应的 Carbon 实体
        if ($datetime) {
            return new Carbon($datetime);
        } else {
        // 否则使用用户注册时间
            return $this->created_at;
        }
    }
}
```

此时意识到一个问题，我们一直在重复哈希表和哈希字段的命名代码，为了提高代码的可维护性，我们需将其抽出为可复用的方法`getHashFromDateString`和`getHashField`，并重构整个 LastActivedAtHelper 类：

_app/Models/Traits/LastActivedAtHelper.php_

    <?php

    namespace App\Models\Traits;

    use Redis;
    use Carbon\Carbon;

    trait LastActivedAtHelper
    {
        // 缓存相关
        protected $hash_prefix = 'larabbs_last_actived_at_';
        protected $field_prefix = 'user_';

        public function recordLastActivedAt()
        {
            // 获取今日 Redis 哈希表名称，如：larabbs_last_actived_at_2017-10-21
            $hash = $this->getHashFromDateString(Carbon::now()->toDateString());

            // 字段名称，如：user_1
            $field = $this->getHashField();

            // 当前时间，如：2017-10-21 08:35:15
            $now = Carbon::now()->toDateTimeString();

            // 数据写入 Redis ，字段已存在会被更新
            Redis::hSet($hash, $field, $now);
        }

        public function syncUserActivedAt()
        {
            // 获取昨日的哈希表名称，如：larabbs_last_actived_at_2017-10-21
            $hash = $this->getHashFromDateString(Carbon::yesterday()->toDateString());

            // 从 Redis 中获取所有哈希表里的数据
            $dates = Redis::hGetAll($hash);

            // 遍历，并同步到数据库中
            foreach ($dates as $user_id => $actived_at) {
                // 会将 `user_1` 转换为 1
                $user_id = str_replace($this->field_prefix, '', $user_id);

                // 只有当用户存在时才更新到数据库中
                if ($user = $this->find($user_id)) {
                    $user->last_actived_at = $actived_at;
                    $user->save();
                }
            }

            // 以数据库为中心的存储，既已同步，即可删除
            Redis::del($hash);
        }

        public function getLastActivedAtAttribute($value)
        {
            // 获取今日对应的哈希表名称
            $hash = $this->getHashFromDateString(Carbon::now()->toDateString());

            // 字段名称，如：user_1
            $field = $this->getHashField();

            // 三元运算符，优先选择 Redis 的数据，否则使用数据库中
            $datetime = Redis::hGet($hash, $field) ? : $value;

            // 如果存在的话，返回时间对应的 Carbon 实体
            if ($datetime) {
                return new Carbon($datetime);
            } else {
            // 否则使用用户注册时间
                return $this->created_at;
            }
        }

        public function getHashFromDateString($date)
        {
            // Redis 哈希表的命名，如：larabbs_last_actived_at_2017-10-21
            return $this->hash_prefix . $date;
        }

        public function getHashField()
        {
            // 字段名称，如：user_1
            return $this->field_prefix . $this->id;
        }
    }

### 第二步。页面显示

在 『注册于』 区块下新增 『最后活跃』，因返回的值为 Carbon 实体，故我们可用`diffForHumans()`来输出用户友好的时间：

_resources/views/users/show.blade.php_

```
.
.
.

        <h5><strong>注册于</strong></h5>
        <p>{{ $user->created_at->diffForHumans() }}</p>
        <hr>
        <h5><strong>最后活跃</strong></h5>
        <p title="{{  $user->last_actived_at }}">{{ $user->last_actived_at->diffForHumans() }}</p>
      </div>
.
.
.
```

打开个人中心即可看到最后活跃时间：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ro4L9pTZNG.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ro4L9pTZNG.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/h2fATSLWKm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/h2fATSLWKm.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户最后活跃时间"
```



