## 基础布局

在教程开始之前，我们需要为我们的项目构建一个基础的页面布局，布局文件统一存放在`resources/views/layouts`文件夹中，布局涉及的文件如下：

* app.blade.php —— 主要布局文件，项目的所有页面都将继承于此页面；
* \_header.blade.php —— 布局的头部区域文件，负责顶部导航栏区块；
* \_footer.blade.php —— 布局的尾部区域文件，负责底部导航区块；

## 主要布局文件

我们先创建主要布局文件：

_resources/views/layouts/app.blade.php_

```
<!DOCTYPE html>
<html lang="{{ app()->getLocale() }}">

<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- CSRF Token -->
  <meta name="csrf-token" content="{{ csrf_token() }}">

  <title>@yield('title', 'LaraBBS') - Laravel 进阶教程</title>

  <!-- Styles -->
  <link href="{{ mix('css/app.css') }}" rel="stylesheet">

</head>

<body>
  <div id="app" class="{{ route_class() }}-page">

    @include('layouts._header')

    <div class="container">

      @include('shared._messages')

      @yield('content')

    </div>

    @include('layouts._footer')
  </div>

  <!-- Scripts -->
  <script src="{{ mix('js/app.js') }}"></script>
</body>

</html>
```

`app()->getLocale()`获取的是`config/app.php`中的`locale`选项，因为我们在前面章节中做了修改，所以此选项的值应为`zh-CN`。

```
<meta name="csrf-token" content="{{ csrf_token() }}">
```

`csrf-token`标签是为了方便前端的 JavaScript 脚本获取 CSRF 令牌。

`@yield('title', 'LaraBBS')`继承此模板的页面，如果没有定制`title`区域的话，就会自动使用第二个参数`LaraBBS`作为标题前缀。

`mix('css/app.css')`会根据`webpack.mix.js`的逻辑来生成 CSS 文件链接。

```
<div id="app" class="{{ route_class() }}-page">
```

`route_class()`是我们自定义的辅助方法，我们还需要在`helpers.php`文件中添加此方法：

_app/helpers.php_

```
<?php

function route_class()
{
    return str_replace('.', '-', Route::currentRouteName());
}
```

此方法会将当前请求的路由名称转换为 CSS 类名称，作用是允许我们针对某个页面做页面样式定制。在后面的章节中会用到。

```
@include('layouts._header')
```

加载顶部导航区块的子模板。

```
@yield('content')
```

占位符声明，允许继承此模板的页面注入内容。

```
@include('layouts._footer')
```

加载页面尾部导航区块的子模板。页面的『顶部导航』和『尾部导航』子模板并不存在，接下来由我们来创建这两个模板。

## 顶部导航

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
        <li class="nav-item"><a class="nav-link" href="#">登录</a></li>
        <li class="nav-item"><a class="nav-link" href="#">注册</a></li>
      </ul>
    </div>
  </div>
</nav>
```

注册登录链接我们将在后面章节中修改。

## 底部导航

创建文件：

_resources/views/layouts/\_footer.blade.php_

```
<footer class="footer">
  <div class="container">
    <p class="float-left">
      由 <a href="http://weibo.com/u/1837553744?is_hot=1" target="_blank">Summer</a> 设计和编码 <span style="color: #e27575;font-size: 14px;">❤</span>
    </p>

    <p class="float-right"><a href="mailto:name@email.com">联系我们</a></p>
  </div>
</footer>
```

> 注：请将`Summer`修改为你自己的常用名和链接，为自己的作品署名。

## 消息提醒

_resources/views/shared/\_messages.blade.php_

```
@foreach (['danger', 'warning', 'success', 'info'] as $msg)
  @if(session()->has($msg))
    <div class="flash-message">
      <p class="alert alert-{{ $msg }}">
        {{ session()->get($msg) }}
      </p>
    </div>
  @endif
@endforeach
```

后面我们往闪存里写入：

```
session()->flash('success', 'This is a success alert—check it out!');
session()->flash('danger', 'This is a danger alert—check it out!');
session()->flash('warning', 'This is a warning alert—check it out!');
session()->flash('info', 'This is a info alert—check it out!');
即可在顶部导航栏下显示对应状态的消息提醒：
```

即可在顶部导航栏下显示对应状态的消息提醒：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/R0iWPwvfOi.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/1/R0iWPwvfOi.png!large)

## 首页展示

### 1. 创建控制器

我们将使用控制器`PagesController`来处理所有自定义页面的逻辑，并使用`root()`方法来处理首页的展示。接下来执行以下命令新建控制器：

```
$ php artisan make:controller PagesController
```

将会生成以下文件：

_app/Http/Controllers/PagesController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PagesController extends Controller
{
    //
}
```

我们新增`root()`方法：

_app/Http/Controllers/PagesController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PagesController extends Controller
{
    public function root()
    {
        return view('pages.root');
    }
}
```

### 2. 视图

控制器`root()`方法中加载了视图`pages.root`，目前我们还没有此视图文件，前往创建：

_resources/views/pages/root.blade.php_

```
@extends('layouts.app')
@section('title', '首页')

@section('content')
  <h1>这里是首页</h1>
@stop
```

Laravel 自带一个主页视图`welcome.blade.php`，既然我们已经自定义了主页视图，即可将废弃的主页视图删除：

```
$ rm resources/views/welcome.blade.php
```

### 3. 绑定路由

接下来绑定下路由，将`web.php`里的内容替换为以下：

_routes/web.php_

```
<?php

Route::get('/', 'PagesController@root')->name('root');
```

打开浏览器尝试访问：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/MFJPGrQhjc.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/MFJPGrQhjc.png!large)

这是因为我们在`resources/views/layouts/app.blade.php`中使用`mix()`方法，而我们还未运行 Laravel Mix 进行编译，找不到`mix-manifest.json`文件，所以报错，没事接下来我们来解决这个问题。

## 样式调整

### 运行 Laravel Mix

Laravel Mix 一款前端任务自动化管理工具，使用了工作流的模式对制定好的任务依次执行。Mix 提供了简洁流畅的 API，让你能够为你的 Laravel 应用定义 Webpack 编译任务。Mix 支持许多常见的 CSS 与 JavaScript 预处理器，通过简单的调用，你可以轻松地管理前端资源。

使用 Mix 很简单，首先你需要使用以下命令安装 npm 依赖即可。我们将使用 Yarn 来安装依赖，在这之前，因为国内的网络原因，我们还需为 Yarn 配置安装加速：

```
$ yarn config set registry https://registry.npm.taobao.org
```

使用 Yarn 安装依赖：

```
$ yarn install
```

安装成功后，运行以下命令即可：

```
$ npm run watch-poll
```

`watch-poll`会在你的终端里持续运行，监控`resources`文件夹下的资源文件是否有发生改变。在`watch-poll`命令运行的情况下，一旦资源文件发生变化，Webpack 会自动重新编译。

> 注意：在后面的课程中，我们需要保证`npm run watch-poll`一直处在执行状态中。

正常运行的界面应类似：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/pEJ5QSU418.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/pEJ5QSU418.png!large)

此时再次刷新即可看到页面正常显示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/cTabuR5KxC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/cTabuR5KxC.png!large)

样式有点奇怪，接下来我们优化下。

### 优化页首

_resources/sass/app.scss_

```
// Variables
@import 'variables';

// Bootstrap
@import '~bootstrap/scss/bootstrap';

/* universal */

body {
  font-family: Helvetica, "Microsoft YaHei", Arial, sans-serif;
  font-size: 14px;
}

/* header */

.navbar-static-top {
  border-color: #e7e7e7;
  background-color: #fff;
  box-shadow: 0px 1px 11px 2px rgba(42, 42, 42, 0.1);
  border-top: 4px solid #00b5ad;
  border-bottom: 1px solid #e8e8e8;
  margin-bottom: 40px;
  margin-top: 0px;
}
```

设置了全局字体，还有顶部导航栏的阴影，效果如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/DIMvEABM8h.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/DIMvEABM8h.png!large)

### 页脚固定

_resources/sass/app.scss_

```
.
.
.

/* Sticky footer styles */
html {
  position: relative;
  min-height: 100%;
}

body {
  /* Margin bottom by footer height */
  margin-bottom: 60px;
}

.footer {
  position: absolute;
  bottom: 0;
  width: 100%;
  /* Set the fixed height of the footer here */
  height: 60px;
  background-color: #000;

  .container {
    padding-right: 15px;
    padding-left: 15px;

    p {
      margin: 19px 0;
      color: #c1c1c1;

      a {
        color: inherit;
      }
    }
  }
}
```

页脚效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/fY4jl03EgX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/fY4jl03EgX.png!large)

至此，我们完成了基础页面结构的构建。

## Git 代码版本控制

接着让我们将该文件加入到版本控制中：

```
$ git add -A
$ git commit -m "基础页面结构"
```



