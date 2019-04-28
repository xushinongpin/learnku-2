## 邮箱认证

从产品设计上讲，『邮箱认证』能让我们有效地检验用户邮箱的真实性，后续网站可以利用这些真实邮箱来联系上用户，例如评论触发邮件通知，或者重要信件等。

另一方面，『邮箱认证』也会对不良用户起到很好的抑制，此类用户注册后会在网站上创建大量垃圾内容，认证其邮箱，提高了注册用户的难度，有效提高网站内容的品质。

我们将只允许邮箱认证通过的用户使用网站，未认证用户会被引导进入验证邮箱页面。

『邮箱认证』工作机制一般分两步：

1. 发送认证邮件 —— 将附带认证信息的『认证链接』发送到用户邮箱里；
2. 检测认证链接 —— 用户打开邮件，点击认证链接进入网站，程序检测 URL 中认证参数的合法性，并渲染对应的页面。

以上流程非常通用，Laravel 默认自带了这个功能，我们可以很方便地进行集成。

屡一下产品思路：

1. 用户注册成功后，给用户发送一个认证邮件；
2. 用户登录状态下，如邮箱未认证，重定向到提醒验证邮箱的页面中。

接下来让我们一起来动手开发吧。

## 修改模型存放位置

Laravel 为我们生成了用户模型文件 app/User.php ，默认情况下，Laravel 会将生成的模型文件放置在`app`文件夹下。为了遵循 MVC 的开发范式，本教程中将统一使用`app/Models`文件夹来放置所有的模型文件。现在让我们先来创建一个`app/Models`文件夹，并将`User.php`文件放置到其中。

```
$ mkdir app/Models
$ mv app/User.php app/Models/User.php
```

在执行完这一步的操作之后，我们还需要执行下面这两个操作：

1、修改 User.php 文件，更改 namespace 为我们新创建的文件夹路径：

_app/Models/User.php_

```
<?php

namespace App\Models;
.
.
.
```

2、编辑器全局搜索`App\User`替换为`App\Models\User`，在 Sublime Text 中可使用快捷键`shift + cmd(ctrl) + f`来进行全局搜索替换的操作。

完成之后，点击右下角的`Replace`按钮替换全部：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/1/usCFkj57bx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/1/usCFkj57bx.png!large)

因为上面的文件改动较大，因此我们需要进行一次 Git 提交，该改动的代码进行保存。

```
$ git add -A
$ git commit -m "移动 User 模型到 app/models 目录"
```

## 修改 User 模型

接下来我们将修改 User 模型，将 Laravel 自带的邮箱认证功能集成到我们的程序中。

_app/Models/User.php_

```
<?php

namespace App\Models;

use Illuminate\Notifications\Notifiable;
use Illuminate\Auth\MustVerifyEmail as MustVerifyEmailTrait;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Contracts\Auth\MustVerifyEmail as MustVerifyEmailContract;

class User extends Authenticatable implements MustVerifyEmailContract
{
    use Notifiable, MustVerifyEmailTrait;

    protected $fillable = [
        'name', 'email', 'password',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];
}
```

**代码详解：**

```
use Notifiable, MustVerifyEmailTrait;
```

加载使用`MustVerifyEmail`trait，打开`vendor/laravel/framework/src/Illuminate/Auth/MustVerifyEmail.php`文件，可以看到以下三个方法：

* `hasVerifiedEmail()`
  检测用户 Email 是否已认证；
* `markEmailAsVerified()`
  将用户标示为已认证；
* `sendEmailVerificationNotification()`
  发送 Email 认证的消息通知，触发邮件的发送。

得益于 PHP 的 trait 功能，User 模型在`use`以后，即可使用以上三个方法。

```
class User extends Authenticatable implements MustVerifyEmailContract
```

可以打开`vendor/laravel/framework/src/Illuminate/Contracts/Auth/MustVerifyEmail.php`，可以看到此文件为 PHP 的接口类，继承此类将确保 User 遵守契约，拥有上面提到的三个方法。

## 发送认证邮件

接下来我们来开发用户注册成功后，发送认证邮件的功能。目前我们使用了 Laravel 自带的`RegisterController`，控制器通过加载`Illuminate\Foundation\Auth\RegistersUsers`trait 来引入框架的注册功能，此时我们打开此 trait 来翻阅源码并定位到`register(Request $request)`方法：

    public function register(Request $request)
        {
            // 检验用户提交的数据是否有误
            $this->validator($request->all())->validate();

            // 创建用户同时触发用户注册成功的事件，并将用户传参
            event(new Registered($user = $this->create($request->all())));

            // 登录用户
            $this->guard()->login($user);

            // 调用钩子方法 `registered()` 
            return $this->registered($request, $user)
                            ?: redirect($this->redirectPath());
        }

此方法处理了用户提交表单后的逻辑，我们把重点放在`event(new Registered($user = $this->create($request->all())));`，这里使用了 Laravel 的事件系统，触发了`Registered`事件。

打开`app/Providers/EventServiceProvider.php`文件，此文件的`$listen`属性里我们可以看到注册了`Registered`事件的监听器：

```
protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];
```

打开`SendEmailVerificationNotification`类，阅读其源码：

_vendor/laravel/framework/src/Illuminate/Auth/Listeners/SendEmailVerificationNotification.php_

```
<?php

namespace Illuminate\Auth\Listeners;

use Illuminate\Auth\Events\Registered;
use Illuminate\Contracts\Auth\MustVerifyEmail;

class SendEmailVerificationNotification
{
    /**
     * 处理事件
     *
     * @param  \Illuminate\Auth\Events\Registered  $event
     * @return void
     */
    public function handle(Registered $event)
    {
        // 如果 user 是继承于 MustVerifyEmail 并且还未激活的话
        if ($event->user instanceof MustVerifyEmail && ! $event->user->hasVerifiedEmail()) {
            // 发送邮件认证消息通知（认证邮件）
            $event->user->sendEmailVerificationNotification();
        }
    }
}
```

可以看出 Laravel 默认已经为我们设置了邮件发送的逻辑，接下来我们来测试一下。

## 开始测试

测试之前，我们先设置下邮件发送到 log 中，以便后面的测试：

_.env_

```
.
.
.
MAIL_DRIVER=log
.
.
.
```

修改成功后浏览器访问[http://larabbs.test/register](http://larabbs.test/register)，填写表单并注册一个测试用户：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/EEYVzxtJsv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/EEYVzxtJsv.png!large)

注册成功后，打开日志存放目录`storage/logs`，打开最近一天的`.log`文件，在文件最尾部应可见类似以下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/5IXFfc6xMm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/5IXFfc6xMm.png!large)

邮件成功发送，复制以上的激活链接，找个地方记录起来，先不要浏览器访问，后续课程会用到。

## 强制用户认证

我们希望用户认证邮箱后，才能使用网站。首先我们来验证下当前登录的用户是否是已经认证过的，做个小测试：

_app/Http/Controllers/PagesController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PagesController extends Controller
{
    public function root()
    {
        dd(\Auth::user()->hasVerifiedEmail());
        return view('pages.root');
    }
}
```

刷新页面可以看到：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/AL3csrxpcm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/AL3csrxpcm.png!large)

`false`代表着还未激活，跟我们预想的一样，接下来我们检出还原`PagesController`：

```
$ git checkout app/Http/Controllers/PagesController.php
```

接下来我们将使用[Laravel 中间件](https://learnku.com/docs/laravel/5.7/middleware)来过滤用户的所有请求，如果用户未认证的话，就跳转到邮件认证提醒的页面中。中间件是 Laravel 中开发经常使用的功能，此处我们先混个脸熟，后面章节我们会有详细讲解的篇幅。

可以使用以下命令来新建一个中间件：

```
$ php artisan make:middleware EnsureEmailIsVerified
```

打开生成的文件并代入以下内容：

_app/Http/Middleware/EnsureEmailIsVerified.php_

```
<?php

namespace App\Http\Middleware;

use Closure;

class EnsureEmailIsVerified
{
    public function handle($request, Closure $next)
    {
        // 三个判断：
        // 1. 如果用户已经登录
        // 2. 并且还未认证 Email
        // 3. 并且访问的不是 email 验证相关 URL 或者退出的 URL。
        if ($request->user() &&
            ! $request->user()->hasVerifiedEmail() &&
            ! $request->is('email/*', 'logout')) {

            // 根据客户端返回对应的内容
            return $request->expectsJson()
                        ? abort(403, 'Your email address is not verified.')
                        : redirect()->route('verification.notice');
        }

        return $next($request);
    }
}
```

接下来注册中间件，注册的时机确保在`StartSession`后面即可：

_app/Http/Kernel.php_

```
<?php
.
.
.
class Kernel extends HttpKernel
{
    .
    .
    .

    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            // \Illuminate\Session\Middleware\AuthenticateSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            \App\Http\Middleware\EnsureEmailIsVerified::class,      // <<--- 只需添加这一行
        ],

        'api' => [
            'throttle:60,1',
            'bindings',
        ],
    ];
    .
    .
    .
}
```

刷新页面，即可看到认证提醒，并且除了我们上面代码中设置的 URL 外都会进入此页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/rpRpt9oCa2.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/rpRpt9oCa2.png!large)

内置邮箱认证还有个小功能，当你点击点击多次『重新发送 Email』后，系统会自动做限额处理，你可以在`VerificationController`中配置相应的信息：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/UyB5kka1l0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/UyB5kka1l0.png!large)

## 最后测试

找到我们在日志里获取到的 URL ，黏贴浏览器里访问，我们将可以再次访问到首页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/XyPdDxxpsH.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/21/1/XyPdDxxpsH.png!large)

## Git 代码版本控制

接着让我们将本次更改纳入版本控制中：

```
$ git add -A
$ git commit -m "邮箱认证"
```



