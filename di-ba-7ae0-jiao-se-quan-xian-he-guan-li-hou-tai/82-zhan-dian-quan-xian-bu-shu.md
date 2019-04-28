## 部署权限

上一章节中，我们安装和初始化了多角色权限管理方案，接下来我们要将权限控制部署到整个项目中。

主要由以下几个地方需要权限控制：

* 拥有
  `manage_contents`
  权限的用户允许管理站点内所有话题和回复，包括编辑和删除动作；
* Horizon 的控制面板，只有
  `站长`
  才有权限查看。

## 1. 内容管理权限

拥有`manage_contents`权限的用户允许管理站点内所有话题和回复，听起来蛮复杂，事实上，得益于 Laravel 授权策略灵活的授权机制，我们只需要几行代码就可以实现。

我们将使用授权策略的[策略过滤器](https://learnku.com/docs/laravel/5.7/authorization#policy-filters)机制来实现统一授权的目的。我们只需在策略中定义一个`before()`方法。`before`方法会在策略中其它所有方法之前执行，这样提供了一种全局授权的方案。在`before`方法中存在三种类型的返回值：

* 返回
  `true`
  是直接通过授权；
* 返回
  `false`
  ，会拒绝用户所有的授权；
* 如果返回的是 null，则通过其它的策略方法来决定授权通过与否。

代码生成器所生成的授权策略，都会统一继承`App\Policies\Policy`基类，这样我们只需要在基类的`before()`方法里做下角色权限判断即可作用到所有的授权类：

_app/Policies/Policy.php_

```
<?php

namespace App\Policies;

use Illuminate\Auth\Access\HandlesAuthorization;

class Policy
{
    use HandlesAuthorization;

    public function before($user, $ability)
    {
        // 如果用户拥有管理内容的权限的话，即授权通过
        if ($user->can('manage_contents')) {
            return true;
        }
    }
}
```

## 2. 用户切换工具 sudo-su

我们现在有一个问题，加上角色权限判断以后，我们需要切换多个用户来测试，频繁地退出和登录用户是一个费时的事情，接下来我们将使用[sudo-su](https://github.com/viacreative/sudo-su)用户切换工具，来提高我们的效率。

### 1. 安装

使用 Composer 安装：

```
$ composer require "viacreative/sudo-su:~1.1"
```

### 2. 添加 Provider

我们只在开发环境中加载此扩展包：

_app/Providers/AppServiceProvider.php_

```
<?php
.
.
.
class AppServiceProvider extends ServiceProvider
{
    .
    .
    .
    public function register()
    {
        if (app()->isLocal()) {
            $this->app->register(\VIACreative\SudoSu\ServiceProvider::class);
        }
    }
}
```

### 3. 发布资源文件

运行以下命令：

```
$ php artisan vendor:publish --provider="VIACreative\SudoSu\ServiceProvider"
```

会生成：

* `/public/sudo-su`
  前端 CSS 资源存放文件夹；
* `config/sudosu.php`
  配置信息文件。

### 4. 修改配置文件

替换配置文件为以下：

_config/sudosu.php_

```
<?php

return [

    // 允许使用的顶级域名
    'allowed_tlds' => ['dev', 'local', 'test'],

    // 用户模型
    'user_model' => App\Models\User::class
];
```

Sudosu 为了避免开发者在生产环境下误开启此功能，在配置选项`allowed_tlds`里做了域名后缀的限制，tld 为 Top Level Domain 的简写。此处因我们的项目域名为`larabbs.test`，故将`test`域名后缀添加到`allowed_tlds`数组中。

### 5. 模板植入

一般我们是在网页里使用 Sudosu，在主要布局模板中的 Scripts 区块上写入模板调用代码：

_resources/views/layouts/app.blade.php_

```
.
.
.
  @if (app()->isLocal())
    @include('sudosu::user-selector')
  @endif

  <!-- Scripts -->
.
.
.
```

此刻刷新页面，即可看到右下角的用户账号切换按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/eEoJYlOrdN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/eEoJYlOrdN.png!large)

### 6. 测试下效果

接下来我们按照以下流程进行测试：

1. 登录 ID 为 1，用户 Summer，并访问非 Summer 发布的话题，检测是否能看到删除按钮；
2. 登录 ID 为 2 的用户，访问其他人发布的话题，检测是否能看到删除按钮；
3. 登录除了 ID 为 1 和 2 的用户，访问其他人发布的话题，检测是否能看到删除按钮。

我们只对 ID 为 1 和 2 的用户做了角色授权，所以其他用户应该没有管理内容的权限：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Yem7ShZbuX.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Yem7ShZbuX.gif!large)

有了这个账号切换按钮，测试用户权限方便多了。

## Horizon 访问权限

Horizon 控制面板页面的路由是`/horizon`，默认只能在`local`环境中访问仪表盘。我们可以使用`Horizon::auth`方法定义更具体的访问策略。`auth`方法能够接受一个回调函数，此回调函数需要返回`true`或`false`，从而确认当前用户是否有权限访问 Horizon 仪表盘。接下来我们定义`/horizon`的访问权限，只有`站长`才有权限查看：

_app/Providers/AuthServiceProvider.php_

```
<?php
.
.
.
class AuthServiceProvider extends ServiceProvider
{
    .
    .
    .
    public function boot()
    {
        $this->registerPolicies();

        \Horizon::auth(function ($request) {
            // 是否是站长
            return \Auth::user()->hasRole('Founder');
        });
    }
}
```

测试一下，一般用户访问会返回 403 报错页面，只有当我们使用 ID 为 1 的 Summer 用户访问时，才能顺利显示页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ZXhtWpCQsA.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ZXhtWpCQsA.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "站点权限部署"
```



