## 问题描述

现代化的浏览器，会对静态文件进行缓存，静态文件在本课程的范畴内，指的是`.css`、`.js`后缀的文件。这是一个浏览器的优化功能，极大地加快了网页的加载速度，但是在我们日常开发和维护中，有时候会造成混淆。

开发时，你明明修改了样式，但是刷新浏览器却看不见变化，然后你就来回不断地修改你的样式文件、重新编译、做各种测试，浏览器页面仍然一成不变。直到你重新刷新好多次，或者修改样式文件名称时，才恍然大悟，原来是浏览器缓存了。

## 解决方案

Laravel Mix 给出的方案是为每一次的文件修改做哈希处理。只要文件修改，哈希值就会变，提醒客户端需要重新加载文件，很巧妙地解决了我们的问题。我们只需要对`webpack.mix.js`稍作修改，即可使用此功能：

_webpack.mix.js_

```
const mix = require('laravel-mix');

mix.js('resources/js/app.js', 'public/js')
   .sass('resources/sass/app.scss', 'public/css').version();
```

以上可看出，我们只是增加了`version()`函数的调用，其他未做修改。

在我们的全局模板中，我们已经使用了动态的加载样式代码，此处无需修改。贴上代码来看下，注意以下`css`和`js`加载的地方：

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

`mix()`方法与`webpack.mix.js`文件里的逻辑遥相呼应。接下来刷新页面，查看页面源代码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/LG1VU5JBRG.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/16/1/LG1VU5JBRG.png!large)

另外的，线上环境中，我们一般会使用 CDN 服务器来加载静态文件，以达到优化网页加载速度的效果。CDN 也会遇到像浏览器缓存类似的问题，Laravel Mix 的`version()`同时也是一个很好解决文件版本变更的方案。

## Git 代码版本控制

接着让我们将本次更改纳入版本控制中：

```
$ git add -A
$ git commit -m "静态文件浏览器缓存问题"
```



