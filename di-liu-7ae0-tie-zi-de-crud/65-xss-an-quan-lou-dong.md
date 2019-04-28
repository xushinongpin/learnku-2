## XSS

目前我们的项目中存在非常严重的 XSS 安全漏洞。作为一个合格的 Web 开发工程师，必须遵循一个安全原则：

> 永远不要信任用户提交的数据。

我们在创建话题时对用户提交的内容字段`body`太过信任，没有对其做过滤处理，即原封不动的显示在页面上，这是非常危险的。

XSS 也称跨站脚本攻击 \(Cross Site Scripting\)，恶意攻击者往 Web 页面里插入恶意 JavaScript 代码，当用户浏览该页之时，嵌入其中 Web 里面的 JavaScript 代码会被执行，从而达到恶意攻击用户的目的。

一种比较常见的 XSS 攻击是 Cookie 窃取。我们都知道网站是通过 Cookie 来辨别用户身份的，一旦恶意攻击者能在页面中执行 JavaScript 代码，他们即可通过 JavaScript 读取并窃取你的 Cookie，拿到你的 Cookie 以后即可伪造你的身份登录网站。（扩展阅读 ——[IBM 文档库：跨站点脚本攻击深入解析](https://www.ibm.com/developerworks/cn/rational/08/0325_segal/)）

有两种方法可以避免 XSS 攻击：

* 第一种，对用户提交的数据进行过滤；
* 第二种，Web 网页显示时对数据进行特殊处理，一般使用
  `htmlspecialchars()`
  输出。

Laravel 的 Blade 语法`{{ }}`会自动调用 PHP`htmlspecialchars`函数来避免 XSS 攻击。但是因为我们支持 WYSIWYG 编辑器，我们使用的是`{!! !!}`来打印用户提交的话题内容：

```
<div class="topic-body">
    {!! $topic->body !!}
</div>
```

Blade 的`{!! !!}`语法是直接数据，不会对数据做任何处理。在我们这种场景下，因为业务逻辑的特殊性，第二种方法不适用，我们将使用第一种方法，对用户提交的数据进行过滤来避免 XSS 攻击。

## 布置舞台

编辑器 Simditor 默认为我们的内容提供转义，我们可以使用以下这段代码来测试：

```
<script>alert('存在 XSS 安全威胁！')</script>
```

[![](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/AEAy5WAiGI.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/AEAy5WAiGI.png)

提交表单后，渲染结果为安全的、被转义的内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/Zd4bKw10nO.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/Zd4bKw10nO.png)

很多同学会大意地认为前端页面有 WYSIWYG 为我们做转义，就是安全的，事实并不是这样的。

现实生活中，不良用户不一定是网页上向服务器提交内容，有太多的工具可以执行此操作，门槛非常低，例如接下来我们演示在 Chrome 的开发者工具栏里直接提交数据到服务器上：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/E4jK7kGkQF.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/E4jK7kGkQF.gif!large)

直接就被注入了，使用数据库里的查看下提交的内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/SCPP0BVxWv.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/SCPP0BVxWv.gif!large)

黏贴进控制台的代码如下：

```
$ fetch("http://larabbs.test/topics", {"headers":{"content-type":"application/x-www-form-urlencoded","upgrade-insecure-requests":"1"},"body":"_token=AGL1jSjzX152b71UEAQiTzwbYdRGYnECRI17WRiG&title=dangerous%20content+&category_id=2&body=%3Cscript%3Ealert%28%27%E5%AD%98%E5%9C%A8%20XSS%20%E5%AE%89%E5%85%A8%E5%A8%81%E8%83%81%EF%BC%81%27%29%3C%2Fscript%3E","method":"POST","mode":"cors"});
```

你需要将`AGL1jSjzX152b71UEAQiTzwbYdRGYnECRI17WRiG`这一行替换为你的 Token ，Chrome 浏览器里右键选择『查看页面源码』即可找到：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/AxUlswnXkI.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/AxUlswnXkI.png!large)

为了方便测试，我们在编辑器中注释掉 Simditor 的代码调用（快捷键 Ctrl + /）：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/ZkgaewUaY7.png!large)](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/ZkgaewUaY7.png!large)

打开新建话题页面，填入测试代码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/GNLb9F8Vz1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/GNLb9F8Vz1.png!large)

点击提交，出现弹框表明我们已经被 XSS 注入，用户可以很轻易地在我们的网页上运行 JavaScript 脚本：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/PDNpYp3Tdx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/PDNpYp3Tdx.png!large)

接下来我们一起解决此问题。

## HTMLPurifier

PHP 一个比较合理的解决方案是 HTMLPurifier 。[HTMLPurifier](http://htmlpurifier.org/)本身就是一个独立的项目，运用『白名单机制』对 HTML 文本信息进行 XSS 过滤。这种过滤机制可以有效地防止各种 XSS 变种攻击。只通过我们认为安全的标签和属性，对于未知的全部过滤。

『白名单机制』指的是使用配置信息来定义『HTML 标签』、『标签属性』和『CSS 属性』数组，在执行`clean()`方法时，只允许配置信息『白名单』里出现的元素通过，其他都进行过滤。

如配置信息：

```
'HTML.Allowed' => 'div,em,a[href|title|style],ul,ol,li,p[style],br',
'CSS.AllowedProperties'    => 'font,font-size,font-weight,font-style,font-family',
```

当用户提交时：

```
<a someproperty="somevalue" href="http://example.com" style="color:#ccc;font-size:16px">
    文章内容<script>alert('Alerted')</script>
</a>
```

会被解析为：

```
<a href="http://example.com" style="font-size:16px">
    文章内容
</a>
```

以下内容因为未指定会被过滤：

1. `someproperty`
   未指定的 HTML 属性
2. `color`
   未指定的 CSS 属性
3. `script`
   未指定的 HTML 标签

## HTMLPurifier for Laravel 5

[HTMLPurifier for Laravel](https://github.com/mewebstudio/Purifier)是对 HTMLPurifier 针对 Laravel 框架的一个封装。本章节中，我们将使用此扩展包来对用户内容进行过滤。

### 1. 安装 HTMLPurifier for Laravel 5

使用 Composer 安装：

```
$ composer require "mews/purifier:~2.0"
```

### 2. 配置 HTMLPurifier for Laravel 5

命令行下运行

```
$ php artisan vendor:publish --provider="Mews\Purifier\PurifierServiceProvider"
```

请将配置信息替换为以下:

_config/purifier.php_

```
<?php

return [
    'encoding'      => 'UTF-8',
    'finalize'      => true,
    'cachePath'     => storage_path('app/purifier'),
    'cacheFileMode' => 0755,
    'settings'      => [
        'user_topic_body' => [
            'HTML.Doctype'             => 'XHTML 1.0 Transitional',
            'HTML.Allowed'             => 'div,b,strong,i,em,a[href|title],ul,ol,ol[start],li,p[style],br,span[style],img[width|height|alt|src],*[style|class],pre,hr,code,h2,h3,h4,h5,h6,blockquote,del,table,thead,tbody,tr,th,td',
            'CSS.AllowedProperties'    => 'font,font-size,font-weight,font-style,margin,width,height,font-family,text-decoration,padding-left,color,background-color,text-align',
            'AutoFormat.AutoParagraph' => true,
            'AutoFormat.RemoveEmpty'   => true,
        ],
    ],
];
```

配置里的`user_topic_body`是我们为话题内容定制的，配合`clean()`方法使用：

```
$topic->body = clean($topic->body, 'user_topic_body');
```

## 开始过滤

一切准备就绪，现在我们只需要在数据入库前进行过滤即可：

_app/Observers/TopicObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Topic;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class TopicObserver
{
    public function saving(Topic $topic)
    {
        $topic->body = clean($topic->body, 'user_topic_body');

        $topic->excerpt = make_excerpt($topic->body);
    }
}
```

浏览器访问话题创建页面，并填入测试内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/5f07739pHB.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/5f07739pHB.png!large)

提交表单后的结果，只剩下正常内容，并且没有弹框：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/ASFgbYG1Te.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/ASFgbYG1Te.png!large)

数据库里可以看到我们提交的 XSS 注入内容已被过滤：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/KE6zNPhgLg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/KE6zNPhgLg.png!large)

HTMLPurifier 把白名单里设定的内容通过，`script`标签未设定，自动过滤掉。

至此 XSS 安全漏洞已修复。最后不要忘了为我们的代码去掉注释：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/HlXtejHLEu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/17/1/HlXtejHLEu.png)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "修复 XSS 注入漏洞"
```



