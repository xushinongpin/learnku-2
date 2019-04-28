## 测试注册功能

点击页面右上角的[注册按钮](http://larabbs.test/register)进入注册页面，并填写表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/5mNpg58Orp.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/5mNpg58Orp.png!large)

点击『注册』按钮提交表单，将会出现下图报错：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/xiyvWpDIIa.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/xiyvWpDIIa.png!large)

这是 Laravel 框架内部集成的异常处理扩展[whoops](https://github.com/filp/whoops)所渲染出来的视图。日常开发中，我们会有大量的机会跟此工具打交道，接下来我们一起来熟悉一下。

## whoops

whoops 是一个非常优秀的 PHP Debug 扩展，它能够使你在开发中快速定位出错的位置。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/SKw1w95jql.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/SKw1w95jql.png!large)

* 区域 1 —— 是错误异常的简介
* 区域 2 —— 是错误发生的位置
* 区域 3 —— 是程序调用堆栈，这里看到脚本调用的顺序
* 区域 4 —— 是一些运行环境的信息，包括：
  * GET Data —— 用户提交的 GET 请求，PHP 超级全局变量
    `$_GET`
    里的内容
  * POST Data —— 表单提交的数据，PHP 超级全局变量
    `$_POST`
    里的内容
  * Files —— 用户上传文件的数据，PHP 超级全局变量
    `$_FILES`
    里的内容
  * Cookies —— 当前用户的 Cookie 信息，PHP 超级全局变量
    `$_COOKIE`
    里的内容
  * Session —— 当前用户会话信息，PHP 超级全局变量
    `$_SESSION`
    里的内容
  * Server/Request Data —— PHP 超级全局变量
    `$_SERVER`
    里的内容
  * Environment Variables —— 项目
    `.env`
    里的内容

## 修复错误

介绍完 whoops ，我们重新回到我们的注册流程中。重点看下报错信息，也就是『区域 1』中的内容：

> Illuminate \ Database \ QueryException \(42S02\)  
> SQLSTATE\[42S02\]: Base table or view not found: 1146 Table 'larabbs.users' doesn't exist \(SQL: select count\(\*\) as aggregate from`users`where`email`= summer@learnku.com\)

`QueryException`是数据库查询语句执行出错异常。上面的简介中，重点在`Table 'larabbs.users' doesn't exist`，数据库`larabbs`中表`users`不存在。

> 注意数据库`larabbs`是在 Homestead 初始化时为我们创建的，我们在 Homestead.yaml 中配置过。

命令行进入 MySQL 终端查看：

```
$ mysql -u homestead -p
```

`mysql`是 MySQL 终端命令的调用名称， 参数`-u`是指定用户，`homestead`是 Homestead 虚拟机中为我们准备好的 MySQL 用户，`-p`参数是告知我们将要为`homestead`用户输入密码。

按回车键以后，命令行将会要求你输入 MySQL 密码，密码在`.env`文件里的`DB_PASSWORD`选项中可以找到，是`secret`，输入密码后按回车，就能进入 MySQL 命令行终端。

在 MySQL 命令行终端里，命令行提示符号为`mysql>`，在接下来的章节中，如果你遇到此命令行提示符，请识别为此命令必须在 MySQL 命令行终端里运行。

接下来查看我们的`larabbs`数据库中，是否有数据库表存在：

```
mysql> use larabbs;
mysql> show tables;
```

执行的结果如下图：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/19/1/LzC33Ek2qj.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/19/1/LzC33Ek2qj.png)

`Empty set`意味着没有任何数据。这是必然的，因为我们还未执行`artisan migrate`命令，接下来我们开始执行数据迁移来创建数据库表结构。

先退出 MySQL 命令行终端：

```
mysql> exit;
```

执行迁移：

```
$ php artisan migrate
```

[![](https://iocaffcdn.phphub.org/uploads/images/201810/26/1/WqFnyFt8lB.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201810/26/1/WqFnyFt8lB.png!large)

执行成功，此时再重新进入 MySQL 命令行终端查看数据库表的情况：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/19/1/i4TS8UkKx4.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/19/1/i4TS8UkKx4.png)

数据库表结构已经创建成功，进入浏览器，刷新错误页面以此来重新提交我们的表单（如果你意外关闭了错误页面，只需要重新填写表单再次提交即可）：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/c4VAFHPqg8.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/c4VAFHPqg8.png!large)

显示 404 页面未找到，我们能看到地址栏链接为[http://larabbs.test/home](http://larabbs.test/home)，因我们自定义了主页，`make:auth`生成的`home`主页文件已经被我们删除，而默认的业务逻辑是在注册成功后，直接跳转到`home`主页，接下来我们需要修改这些逻辑。

在编辑器中使用快捷键`shift + cmd(ctrl) + f`全局搜索，可以看到如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/F6dZpFexZ6.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/F6dZpFexZ6.png!large)

请将上面显示的五处地方的`'/home'`字串修改为`/`，请勿使用全局替换。

修改后效果如以下：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/mq6s3CjnHH.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/21/1/mq6s3CjnHH.png)

## 登录状态

Laravel 默认注册后的用户是已登录的，接下来我们要制作顶部导航栏来响应用户的登录状态：

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
        @guest
          <li class="nav-item"><a class="nav-link" href="{{ route('login') }}">登录</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ route('register') }}">注册</a></li>
        @else
          <li class="nav-item dropdown">
            <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
              <img src="https://iocaffcdn.phphub.org/uploads/images/201709/20/1/PtDKbASVcz.png?imageView2/1/w/60/h/60" class="img-responsive img-circle" width="30px" height="30px">
              {{ Auth::user()->name }}
            </a>
            <div class="dropdown-menu" aria-labelledby="navbarDropdown">
              <a class="dropdown-item" href="">个人中心</a>
              <a class="dropdown-item" href="">编辑资料</a>
              <div class="dropdown-divider"></div>
              <a class="dropdown-item" id="logout" href="#">
                <form action="{{ route('logout') }}" method="POST">
                  {{ csrf_field() }}
                  <button class="btn btn-block btn-danger" type="submit" name="button">退出</button>
                </form>
              </a>
            </div>
          </li>
        @endguest
      </ul>
    </div>
  </div>
</nav>
```

**代码讲解：**

重点看 Blade 的`guest`条件语句：

```
@guest
    <li><a href="{{ route('login') }}">登录</a></li>
    <li><a href="{{ route('register') }}">注册</a></li>
@else
    .
    .
    .
@endguest
```

如果是未登录用户的话，就显示注册和登录按钮，如果是已登录用户的话，即显示用户菜单。

## 登录注册用户

重新打开首页[http://larabbs.test/](http://larabbs.test/)，点击右上角的下拉菜单中的『退出登录』按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/Zf2CEmJcfF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/Zf2CEmJcfF.png!large)

点击右上角[登录](http://larabbs.test/login)按钮，填写上一步注册时使用的邮箱和密码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/wIZHXekwUg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/wIZHXekwUg.png!large)

点击 Login 按钮提交进行登录：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/KKpwIZz8Wr.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/17/1/KKpwIZz8Wr.png!large)

能看到我们用户名和乔布斯老爷子的头像，意味着登录成功。

## Git 版本控制

至此注册登录功能已经完成，接下来把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "修复跳转链接"
```



