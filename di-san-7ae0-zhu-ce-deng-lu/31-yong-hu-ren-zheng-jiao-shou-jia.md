## 用户认证脚手架

Laravel 自带了用户认证功能，我们将利用此功能来快速构建我们的用户中心。

首先执行认证脚手架命令，生成代码：

```
$ php artisan make:auth
```

命令`make:auth`会询问我们是否要覆盖`app.blade.php`，因为我们在前面章节中已经自定义了『主要布局文件』——`app.blade.php`，所以此处输入`no`，如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/f3b0wbZMI0.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/f3b0wbZMI0.png)

使用`git status`来查看文件更改的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/6XSxiwlbDr.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/6XSxiwlbDr.png)

打开`routes/web.php`查看修改了哪些内容：

_routes/web.php_

```
<?php

Route::get('/', 'PagesController@root')->name('root');
Auth::routes();

Route::get('/home', 'HomeController@index')->name('home');
```

可以看到在我们的主页下，多了两个表达式，先看第一个：

```
Auth::routes();
```

此处是 Laravel 的用户认证路由，可以在`vendor/laravel/framework/src/Illuminate/Routing/Router.php`中搜索关键词`LoginController`即可找到定义的地方，以上等同于：

```
// 用户身份验证相关的路由
Route::get('login', 'Auth\LoginController@showLoginForm')->name('login');
Route::post('login', 'Auth\LoginController@login');
Route::post('logout', 'Auth\LoginController@logout')->name('logout');

// 用户注册相关路由
Route::get('register', 'Auth\RegisterController@showRegistrationForm')->name('register');
Route::post('register', 'Auth\RegisterController@register');

// 密码重置相关路由
Route::get('password/reset', 'Auth\ForgotPasswordController@showLinkRequestForm')->name('password.request');
Route::post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail')->name('password.email');
Route::get('password/reset/{token}', 'Auth\ResetPasswordController@showResetForm')->name('password.reset');
Route::post('password/reset', 'Auth\ResetPasswordController@reset')->name('password.update');

// Email 认证相关路由
Route::get('email/verify', 'Auth\VerificationController@show')->name('verification.notice');
Route::get('email/verify/{id}', 'Auth\VerificationController@verify')->name('verification.verify');
Route::get('email/resend', 'Auth\VerificationController@resend')->name('verification.resend');
```

为了更加直观，我们将在`web.php`中使用以上替换`Auth::routes();`。

再来看下面这一行：

```
Route::get('/home', 'HomeController@index')->name('home');
```

我们已经有自己的主页了，不需要再次设置主页，直接删除即可。修改后的路由文件内容如下，请直接替换：

_routes/web.php_

```
<?php

Route::get('/', 'PagesController@root')->name('root');

// 用户身份验证相关的路由
Route::get('login', 'Auth\LoginController@showLoginForm')->name('login');
Route::post('login', 'Auth\LoginController@login');
Route::post('logout', 'Auth\LoginController@logout')->name('logout');

// 用户注册相关路由
Route::get('register', 'Auth\RegisterController@showRegistrationForm')->name('register');
Route::post('register', 'Auth\RegisterController@register');

// 密码重置相关路由
Route::get('password/reset', 'Auth\ForgotPasswordController@showLinkRequestForm')->name('password.request');
Route::post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail')->name('password.email');
Route::get('password/reset/{token}', 'Auth\ResetPasswordController@showResetForm')->name('password.reset');
Route::post('password/reset', 'Auth\ResetPasswordController@reset')->name('password.update');

// Email 认证相关路由
Route::get('email/verify', 'Auth\VerificationController@show')->name('verification.notice');
Route::get('email/verify/{id}', 'Auth\VerificationController@verify')->name('verification.verify');
Route::get('email/resend', 'Auth\VerificationController@resend')->name('verification.resend');
```

## 生成的视图

`make:auth`命令为我们生成了`resources/views/auth`下的四个文件：

| 视图名称 | 说明 |
| :--- | :--- |
| register.blade.php | 注册页面视图 |
| login.blade.php | 登录页面视图 |
| verify.blade.php | 邮箱认证视图 |
| passwords/email.blade.php | 提交邮箱发送邮件的视图 |
| passwords/reset.blade.php | 重置密码的页面视图 |

## 移除无用页面

因为无需使用`make:auth`生成的主页，请运行以下命令删除无用文件：

```
$ rm app/Http/Controllers/HomeController.php
$ rm resources/views/home.blade.php
```

## 顶部导航

顶部导航『注册』和『登录』入口之前用的是两个假链接，现在我们已经有业务逻辑，接下来将链接套进去，并加上登录后的效果：

_resources/views/layouts/\_header.blade.php_

```
<nav class="navbar navbar-expand-lg navbar-light bg-light navbar-static-top">
  <div class="container">
    <!-- Branding Image -->
    <a class="navbar-brand " href="{{ url('/') }}">
      LaraBBS
    </a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
      <!-- Left Side Of Navbar -->
      <ul class="navbar-nav mr-auto">

      </ul>

      <!-- Right Side Of Navbar -->
      <ul class="navbar-nav navbar-right">
        <!-- Authentication Links -->
        <li class="nav-item"><a class="nav-link" href="{{ route('login') }}">登录</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ route('register') }}">注册</a></li>
      </ul>
    </div>
  </div>
</nav>
```

现在通过顶部导航访问登录页面，看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/0GOKGrLPD0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/0GOKGrLPD0.png!large)

## 本地化

可以看到登录表单是英文版本的，打开模板文件，此模板文件是我们刚刚使用`make:auth`命令生成的：

> resources/views/auth/login.blade.php

可以看到很多`__()`函数的调用：

```
<div class="card-header">{{ __('Login') }}</div>
```

```
<label for="password" class="col-md-4 col-form-label text-md-right">{{ __('Password') }}</label>
```

```
{{ __('Remember Me') }}
```

这是 Laravel 提供的本地化特性，使用`__()`函数来辅助实现。按照约定，本地化文件存储在`resources/lang`文件夹中，为 JSON 格式。在`config/app.php`文件中，我们设置了：

```
'locale' => 'zh-CN',
```

对应翻译文件就是`resources/lang/zh-CN.json`，需新建此文件：

_resources/lang/zh-CN.json_

```
{
    "Login": "登录",
    "Password": "密码",
    "Remember Me": "记住我"
}
```

再次刷新登录页面，可以看到翻译的内容，同时红框里也是我们漏下的内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/qkkdE0nrzJ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/qkkdE0nrzJ.png!large)

## 中文语言包

会有很多人会遇到翻译 Laravel 自带模板的问题，所以我们无需自己一个个去翻译，这种通用的问题找找扩展包来处理即可。

我们将使用[Laravel Lang](https://github.com/overtrue/laravel-lang)项目来实现，此项目支持了 52 个国家的语言，使用以下命令安装：

```
$ composer require "overtrue/laravel-lang:~3.0"
```

完成上面的操作后，将项目文件`config/app.php`中的下一行

```
Illuminate\Translation\TranslationServiceProvider::class,  
```

替换为：

```
Overtrue\LaravelLang\TranslationServiceProvider::class,  
```

完成后刷新页面，完美翻译：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/36Bxlq3FLT.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/36Bxlq3FLT.png!large)

点击上图的忘记密码链接，进入忘记密码的视图，也可以看到成功显示中文：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/iqFUWFPcYh.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/iqFUWFPcYh.png!large)

[Laravel Lang](https://github.com/overtrue/laravel-lang)同自定义语言包一样，都是根据`config/app.php`里`locale`的选项来选择语言的。

值得一提的是，如果你想修改扩展包提供的语言文件，可以使用以下命令发布语言文件到项目里：

```
$ php artisan lang:publish zh-CN
```

发布后的语言文件存放于`resources/lang/zh-CN`文件夹。

## Git 版本控制

至此注册登录功能已经完成，接下来把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "生成用户认证代码"
```



