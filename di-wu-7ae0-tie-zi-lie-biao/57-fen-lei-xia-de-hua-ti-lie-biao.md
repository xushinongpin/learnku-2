## 分类下的话题列表

『话题分类』功能有助于话题的归类，方便用户查阅信息，本章节中，我们将开发以下功能：

* 根据分类显示话题列表；
* 列表页面标题定制；
* 增加顶部导航栏；
* 导航栏的选中状态。

## 1. 根据分类列表话题

路由文件新增一行：

_routes/web.php_

```
.
.
.

Route::resource('categories', 'CategoriesController', ['only' => ['show']]);
```

使用命令行创建控制器：

```
$ php artisan make:controller CategoriesController
```

控制器中新增方法`show()`：

_app/Http/Controllers/CategoriesController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Topic;
use App\Models\Category;

class CategoriesController extends Controller
{
    public function show(Category $category)
    {
        // 读取分类 ID 关联的话题，并按每 20 条分页
        $topics = Topic::where('category_id', $category->id)->paginate(20);
        // 传参变量话题和分类到模板中
        return view('topics.index', compact('topics', 'category'));
    }
}
```

接下来在话题列表中为分类添加`categories.show`路由链接：

_resources/views/topics/\_topic\_list.blade.php_

```
.
.
,
            <a class="text-secondary" href="{{ route('categories.show', $topic->category_id) }}" title="{{ $topic->category->name }}">
              <i class="far fa-folder"></i>
              {{ $topic->category->name }}
            </a>
.
.
.
```

我们还需定制列表页面模板，用以标示当前所在的分类，以此来与『所有话题列表页面』区分：

_resources/views/topics/index.blade.php_

```
@extends('layouts.app')

@section('title', '话题列表')

@section('content')

<div class="row mb-5">
  <div class="col-lg-9 col-md-9 topic-list">
    @if (isset($category))
      <div class="alert alert-info" role="alert">
        {{ $category->name }} ：{{ $category->description }}
      </div>
    @endif

    <div class="card ">

.
.
.
```

浏览器访问话题列表页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/DXRuK7kA9A.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/DXRuK7kA9A.png!large)

并点击某个分类的链接进入分类列表页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/NolUYCsUhL.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/NolUYCsUhL.png!large)

样式有些混乱。这是因为我们`app.scss`中使用了在主模板中定义的路由 CSS 类名称：

_resources/views/layouts/app.blade.php_

```
.
.
.

<body>
    <div id="app" class="{{ route_class() }}-page">
.
.
.
```

虽然我们使用同一个模板，但是当前的页面路由名称已变为`categories.show`，对应的 CSS 类名已变为`categories-show-page`。修复方法很简单，样式表中新添加选择器即可：

_resources/sass/app.scss_

```
.
.
.
/* Topic Index Page */
.topics-index-page, .categories-show-page {
.
.
.
```

等样式重新编译后，刷新页面即可看到正常样式：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/aNnNzNszPy.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/aNnNzNszPy.png!large)

## 2. 列表页面标题定制

当我们在『分类话题列表』页面时，我们希望页面的标题里有当前的分类名称：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/EDgOAwl2P3.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/EDgOAwl2P3.png!large)

修改列表页面模板，使用[三元运算符](https://baike.baidu.com/item/%E4%B8%89%E5%85%83%E8%BF%90%E7%AE%97%E7%AC%A6/1394210)来定制`title`区块：

_resources/views/topics/index.blade.php_

```
@extends('layouts.app')

@section('title', isset($category) ? $category->name : '话题列表')

.
.
.
```

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/t0eo7gsisi.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/t0eo7gsisi.png!large)

## 3. 增加顶部导航栏

接下来新增顶部导航栏，让用户能更加方便的查看分类信息：

_resources/views/layouts/\_header.blade.php_

```
.
.
.

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
      <!-- Left Side Of Navbar -->
      <ul class="navbar-nav mr-auto">
        <li class="nav-item active"><a class="nav-link" href="{{ route('topics.index') }}">话题</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 1) }}">分享</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 2) }}">教程</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 3) }}">问答</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 4) }}">公告</a></li>
      </ul>
.
.
.
```

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/S1gITwOWGF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/S1gITwOWGF.png!large)

现在我们可以很方便地通过顶部导航栏来筛选信息，接下来我们需要处理下选中状态。

### 导航的 Active 状态

导航的选中状态样式是通过添加 CSS 类选择器`active`来实现的：

```
<ul class="navbar-nav mr-auto">
  <li class="nav-item active"><a class="nav-link" href="{{ route('topics.index') }}">话题</a></li>
  <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 1) }}">分享</a></li>
  <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 2) }}">教程</a></li>
  <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 3) }}">问答</a></li>
  <li class="nav-item"><a class="nav-link" href="{{ route('categories.show', 4) }}">公告</a></li>
</ul>
```

此样式是 Bootstrap 框架的[导航栏组件](https://getbootstrap.com/docs/4.0/components/navbar)提供。

我们需要通过判断『路由命名』和『路由参数』为导航栏添加`active`类，接下来我们使用一个很方便的类库来辅助我们实现此功能。

使用 Composer 安装[hieu-le/active](https://github.com/letrunghieu/active)：

```
$ composer require "hieu-le/active:~3.5"
```

安装完成后，在模板中使用：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
    <div class="collapse navbar-collapse" id="navbarSupportedContent">
      <!-- Left Side Of Navbar -->
      <ul class="navbar-nav mr-auto">
        <li class="nav-item {{ active_class(if_route('topics.index')) }}"><a class="nav-link" href="{{ route('topics.index') }}">话题</a></li>
        <li class="nav-item {{ active_class((if_route('categories.show') && if_route_param('category', 1))) }}"><a class="nav-link" href="{{ route('categories.show', 1) }}">分享</a></li>
        <li class="nav-item {{ active_class((if_route('categories.show') && if_route_param('category', 2))) }}"><a class="nav-link" href="{{ route('categories.show', 2) }}">教程</a></li>
        <li class="nav-item {{ active_class((if_route('categories.show') && if_route_param('category', 3))) }}"><a class="nav-link" href="{{ route('categories.show', 3) }}">问答</a></li>
        <li class="nav-item {{ active_class((if_route('categories.show') && if_route_param('category', 4))) }}"><a class="nav-link" href="{{ route('categories.show', 4) }}">公告</a></li>
      </ul>
.
.
.
```

上面代码看起来很复杂，太多重复代码，我们来优化下。将重复代码抽出来放到一个辅助函数里：

_app/helpers.php_

```
.
.
.

function category_nav_active($category_id)
{
    return active_class((if_route('categories.show') && if_route_param('category', $category_id)));
}
```

重新修改模板里的调用：

_resources/views/layouts/\_header.blade.php_

```
.
.
.

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
      <!-- Left Side Of Navbar -->
      <ul class="navbar-nav mr-auto">
        <li class="nav-item {{ active_class(if_route('topics.index')) }}"><a class="nav-link" href="{{ route('topics.index') }}">话题</a></li>
        <li class="nav-item {{ category_nav_active(1) }}"><a class="nav-link" href="{{ route('categories.show', 1) }}">分享</a></li>
        <li class="nav-item {{ category_nav_active(2) }}"><a class="nav-link" href="{{ route('categories.show', 2) }}">教程</a></li>
        <li class="nav-item {{ category_nav_active(3) }}"><a class="nav-link" href="{{ route('categories.show', 3) }}">问答</a></li>
        <li class="nav-item {{ category_nav_active(4) }}"><a class="nav-link" href="{{ route('categories.show', 4) }}">公告</a></li>
      </ul>
.
.
.
```

接下来讲解下`active_class`函数的用法，此函数的定义如下：

    /**
     * 如果 $condition 不为 True 即会返回字符串 `active`
     *
     * @param        $condition
     * @param string $activeClass
     * @param string $inactiveClass
     *
     * @return string
     */
    function active_class($condition, $activeClass = 'active', $inactiveClass = '')

如果传参满足指定条件 \(`$condition`\) ，此函数将返回`$activeClass`，否则返回`$inactiveClass`。

此扩展包提供了一批函数让我们更方便的进行`$condition`判断：

1. if\_route \(\) - 判断当前对应的路由是否是指定的路由；
2. if\_route\_param \(\) - 判断当前的 url 有无指定的路由参数。
3. if\_query \(\) - 判断指定的 GET 变量是否符合设置的值；
4. if\_uri \(\) - 判断当前的 url 是否满足指定的 url；
5. if\_route\_pattern \(\) - 判断当前的路由是否包含指定的字符；
6. if\_uri\_pattern \(\) - 判断当前的 url 是否含有指定的字符；

在这里我们用到第 1 和 第 2 。

测试一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/HVTFL2fdet.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/HVTFL2fdet.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "分类话题列表"
```



