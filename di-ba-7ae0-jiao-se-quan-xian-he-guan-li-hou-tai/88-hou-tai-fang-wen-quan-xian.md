## 后台访问权限

开发后台时，不论是`administrator.php`里、模型配置文件中或者站点配置信息里，我们都有设置到`permission`选项，不过一直没有测试其有效性。这个章节我们统一做测试。

1 号用户 Summer 是站长权限，拥有所有权限，所以不需要测试。接下来我们将切换到第 2 号用户管理员角色和第 3 号普通用户角色进行测试。

首先测试 3 号用户，检测是否能进入后台：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/uVU35LL22M.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/uVU35LL22M.gif!large)

3 号用户无法登录后台，被跳转到首页。事实上，根据`administrator.php`里的配置，当`permission`选项判断不通过后，会重定向到`login`页面，`login`页面检测到用户已经登录，就跳转到首页。这个用户体验不好，接下来我们开发一个无权限访问后台的提醒页面。

## 无权限提醒页面

### 1. 新建路由

_routes/web.php_

```
.
.
.

Route::get('permission-denied', 'PagesController@permissionDenied')->name('permission-denied');
```

### 2. Administrator 配置修改

_config/administrator.php_

    <?php

    return array(
    .
    .
    .
        // 当选项 `permission` 权限检测不通过时，会重定向用户到此处设置的路径
        'login_path' => 'permission-denied',
    .
    .
    .
    );

### 3. 新增控制器方法

_app/Http/Controllers/PagesController.php_

```
<?php
.
.
.
class PagesController extends Controller
{
    .
    .
    .

    public function permissionDenied()
    {
        // 如果当前用户有权限访问后台，直接跳转访问
        if (config('administrator.permission')()) {
            return redirect(url(config('administrator.uri')), 302);
        }
        // 否则使用视图
        return view('pages.permission_denied');
    }
}
```

### 4. 视图文件

_resources/views/pages/permission\_denied.blade.php_

```
@extends('layouts.app')
@section('title', '无权限访问')

@section('content')
  <div class="col-md-4 offset-md-4">
    <div class="card ">
      <div class="card-body">
        @if (Auth::check())
          <div class="alert alert-danger text-center mb-0">
            当前登录账号无后台访问权限。
          </div>
        @else
          <div class="alert alert-danger text-center">
            请登录以后再操作
          </div>

          <a class="btn btn-lg btn-primary btn-block" href="{{ route('login') }}">
            <i class="fas fa-sign-in-alt"></i>
            登 录
          </a>
        @endif
      </div>
    </div>
  </div>
@stop
```

### 5. 测试一下

分别使用以下三种用户权限尝试：

1. 未登录用户；
2. 登录无权限用户；
3. 最后是站长访问。

示例：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CebmrHjdSl.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CebmrHjdSl.gif!large)

## 后台部分可见

`administrator.php`配置项`permission`控制的是进入后台的权限，单独的模型文件里`permission`控制的是模型页面的访问权限，如果模型权限未通过，后台菜单项都会隐藏。接下来我们讨论单独模型的权限控制。

我们的 1 号用户 Summer 的角色是站长，拥有所有权限，2 号用户是管理员，只拥有管理内容的权限，没有『用户管理』权限，可以理解为，只有站长才能删除用户、修改用户组权限和修改站点配置。接下来我们使用 2 号用户来访问后台：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/e0JlNZI7oD.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/e0JlNZI7oD.png!large)

Chrome 浏览器会报错`ERR_TOO_MANY_REDIRECTS`，意为太多跳转死循环，页面无法渲染。原因是`administrator.php`中，我们将`home_page`选项设置为`users`页面，当我们使用 2 号用户访问`/admin`时，会自动跳转到`users`页面，`users`页面检测到 2 号用户没有访问权限，遂重定向到后台首页`/admin`中，访问首页又会重定向到`users`中，所以就是死循环。解决的方法很简单，将`home_page`选项改为访问权限较低的页面如`topics`，这个页面是所有进入后台的用户都有权限访问的：

_config/administrator.php_

    <?php

    return array(
    .
    .
    .
        // 用来作为后台主页的菜单条目，由 `use_dashboard` 选项决定，菜单指的是 `menu` 选项
        'home_page' => 'topics',
    .
    .
    .
    );

修改完成后，当我们再次使用 2 号用户访问后台，会跳转到`topics`模型管理页面下，菜单里也只有内容管理子菜单，其他无权限的菜单皆隐藏着：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/zcsFHbTP5I.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/zcsFHbTP5I.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "管理后台权限"
```



