## 功能说明

接下来我们将制作用户的个人中心页面，作为用户的个人信息展示页。在此页面中，我们可以看到该用户发过的帖子，发表的评论等。

## 设置路由

我们使用 Laravel 的[资源控制器](https://learnku.com/docs/laravel/5.7/controllers#resource-controllers)功能，接下来我们先给控制器注册一个资源路由：

_routes/web.php_

```
.
.
.

Route::resource('users', 'UsersController', ['only' => ['show', 'update', 'edit']]);
```

上面代码将等同于：

```
Route::get('/users/{user}', 'UsersController@show')->name('users.show');
Route::get('/users/{user}/edit', 'UsersController@edit')->name('users.edit');
Route::patch('/users/{user}', 'UsersController@update')->name('users.update');
```

可以看到使用`resource`方法不仅节省很多代码，且严格遵循了 RESTful URI 的规范，在后续的开发中，我们会优先选择`resource`路由。

生成的资源路由列表信息如下所示：

| HTTP 请求 | URI | 动作 | 作用 |
| :--- | :--- | :--- | :--- |
| GET | /users/{user} | UsersController@show | 显示用户个人信息页面 |
| GET | /users/{user}/edit | UsersController@edit | 显示编辑个人资料页面 |
| PATCH | /users/{user} | UsersController@update | 处理 edit 页面提交的更改 |

## 创建控制器

接下来我们需要创建一个 UsersController 控制器，这个控制器将负责用户相关的页面和逻辑处理。Laravel 的控制器命名规范统一使用驼峰式大小写和复数形式来命名，在这里我们也应该这么做。一般情况下，我们会使用下面命令来生成控制器：

```
$ php artisan make:controller UsersController
```

让我们来看下`UsersController`文件生成的默认代码：

_app/Http/Controllers/UsersController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UsersController extends Controller
{
    //
}
```

接下来我们增加`show`方法来处理个人页面的展示：

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\User;

class UsersController extends Controller
{
    public function show(User $user)
    {
        return view('users.show', compact('user'));
    }
}
```

1. 第一个修改是引入了`App\Models\User`用户模型，因为将要在`show()`方法中使用到`User`模型，所以我们必须先引用。

2. 接下来我们看下
   `show()`
   方法的声明：

```
public function show(User $user)
```

Laravel 会自动解析定义在控制器方法（变量名匹配路由片段）中的 Eloquent 模型类型声明。在上面代码中，由于`show()`方法传参时声明了类型 —— Eloquent 模型`User`，对应的变量名`$user`会匹配路由片段中的`{user}`，这样，Laravel 会自动注入与请求 URI 中传入的 ID 对应的用户模型实例。

此功能称为[『隐性路由模型绑定』](https://learnku.com/docs/laravel/5.7/routing#%E9%9A%90%E5%BC%8F%E7%BB%91%E5%AE%9A)，是『约定优于配置』设计范式的体现，同时满足以下两种情况，此功能即会自动启用：

1\). 路由声明时必须使用 Eloquent 模型的单数小写格式来作为**路由片段参数**，User 对应`{user}`：

```
Route::get('/users/{user}', 'UsersController@show')->name('users.show');
```

上面路由部分讲过，在使用资源路由`Route::resource('users', 'UsersController');`时，默认已经包含了上面的声明。

2\). 控制器方法传参中必须包含对应的**Eloquent 模型类型**提示，并且是有序的：

```
public function show(User $user)
{
    return view('users.show', compact('user'));
}
```

当请求[http://larabbs.test/users/1](http://larabbs.test/users/1)并且满足以上两个条件时，Laravel 将会自动查找 ID 为 1 的用户并赋值到变量`$user`中，如果数据库中找不到对应的模型实例，会自动生成 HTTP 404 响应，例如此刻我们只注册了 Summer 和 Monkey 用户，数据库里只有两条数据，当我们访问[http://larabbs.test/users/3](http://larabbs.test/users/3)ID 为 3 的用户时：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/aoUNFde8uY.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/aoUNFde8uY.png!large)

> 注：图片中使用[Chrome 开发者工具](https://jingyan.baidu.com/article/20095761c1414acb0721b4bd.html)查看网络请求。

1. 继续看
   `show()`
   方法里的代码：

```
return view('users.show', compact('user'));
```

我们将用户对象变量`$user`通过[`compact`](http://php.net/manual/zh/function.compact.php)方法转化为一个关联数组，并作为第二个参数传递给`view`方法，将变量数据传递到视图中。

`show`方法添加完成之后，在视图中，我们即可直接使用`$user`变量来获取`view`方法传递给视图的用户数据。

由于我们还没有创建用户个人页面，因此这时访问用户页面时会出现如下报错。

```
View [users.show] not found.
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/mNtQwms6no.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/mNtQwms6no.png!large)

## 创建视图

下面让我们来新建一个用户个人页面。

_resources/views/users/show.blade.php_

```
@extends('layouts.app')

@section('title', $user->name . ' 的个人中心')

@section('content')

<div class="row">

  <div class="col-lg-3 col-md-3 hidden-sm hidden-xs user-info">
    <div class="card ">
      <img class="card-img-top" src="https://iocaffcdn.phphub.org/uploads/images/201709/20/1/PtDKbASVcz.png?imageView2/1/w/600/h/600" alt="{{ $user->name }}">
      <div class="card-body">
            <h5><strong>个人简介</strong></h5>
            <p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. </p>
            <hr>
            <h5><strong>注册于</strong></h5>
            <p>January 01 1901</p>
      </div>
    </div>
  </div>
  <div class="col-lg-9 col-md-9 col-sm-12 col-xs-12">
    <div class="card ">
      <div class="card-body">
          <h1 class="mb-0" style="font-size:22px;">{{ $user->name }} <small>{{ $user->email }}</small></h1>
      </div>
    </div>
    <hr>

    {{-- 用户发布的内容 --}}
    <div class="card ">
      <div class="card-body">
        暂无数据 ~_~
      </div>
    </div>

  </div>
</div>
@stop
```

直接读取`$user`对象的属性：

```
{{ $user->name }}
```

这时候我们再访问用户个人页面，便能够看到基本的数据展示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/4pOhtxQIaY.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/4pOhtxQIaY.png!large)

如上图所示，头像、个人简介和注册时间还都是假数据，在下一个章节中我们将主要专注于这些功能。

## Git 代码标记

我们先把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户个人页面原型"
```



