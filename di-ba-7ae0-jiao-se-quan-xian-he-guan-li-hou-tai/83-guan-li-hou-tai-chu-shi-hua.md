## 管理后台

接下来我们将使用[Laravel Administrator](https://github.com/summerblue/administrator)来构建 LaraBBS 管理员后台，管理员可以在后台完成用户、话题和回复的 CURD 操作，还可完成用户权限分配、修改站点设置等任务。

这是一个使用「配置信息」来快速生成管理员后台的开发理念，让你几分钟内就拥有一个可用的后台。这种开发的理念，因为其开发的快速，广受拥戴，以至于各个流行框架都有对应的解决的方案，如 Ruby on Rails 框架的[Active Admin](https://github.com/activeadmin/activeadmin)， Django 框架也有类似的[管理界面](https://docs.djangoproject.com/en/1.11/intro/tutorial02/)。

## 安装

### 1. 使用 Composer 安装：

```
$ composer require "summerblue/administrator:~1.1"
```

### 2. 发布资源文件

运行以下命令来发布资源文件：

```
$ php artisan vendor:publish --provider="Frozennode\Administrator\AdministratorServiceProvider"
```

会生成：

* config/administrator.php —— 配置信息；
* public/packages/summerblue/administrator —— 前端资源文件，这是用来做页面渲染的。

### 3. 配置信息讲解

请将以下内容替换并仔细阅读代码中的注释：

_config/administrator.php_

    <?php

    return array(

        // 后台的 URI 入口
        'uri' => 'admin',

        // 后台专属域名，没有的话可以留空
        'domain' => '',

        // 应用名称，在页面标题和左上角站点名称处显示
        'title' => env('APP_NAME', 'Laravel'),

        // 模型配置信息文件存放目录
        'model_config_path' => config_path('administrator'),

        // 配置信息文件存放目录
        'settings_config_path' => config_path('administrator/settings'),

        /*
         * 后台菜单数组，多维数组渲染结果为多级嵌套菜单。
         *
         * 数组里的值有三种类型：
         * 1. 字符串 —— 子菜单的入口，不可访问；
         * 2. 模型配置文件 —— 访问 `model_config_path` 目录下的模型文件，如 `users` 访问的是 `users.php` 模型配置文件；
         * 3. 配置信息 —— 必须使用前缀 `settings.`，对应 `settings_config_path` 目录下的文件，如：默认设置下，
         *              `settings.site` 访问的是 `administrator/settings/site.php` 文件
         * 4. 页面文件 —— 必须使用前缀 `page.`，如：`page.pages.analytics` 对应 `administrator/pages/analytics.php`
         *               或者是 `administrator/pages/analytics.blade.php` ，两种后缀名皆可
         *
         * 示例：
         *  [
         *      'users',
         *      'E-Commerce' => ['collections', 'products', 'product_images', 'orders'],
         *      'Settings'  => ['settings.site', 'settings.ecommerce', 'settings.social'],
         *      'Analytics' => ['E-Commerce' => 'page.pages.analytics'],
         *  ]
         */
        'menu' => [
            '用户与权限' => [
                'users',
            ],
        ],

        /*
         * 权限控制的回调函数。
         *
         * 此回调函数需要返回 true 或 false ，用来检测当前用户是否有权限访问后台。
         * `true` 为通过，`false` 会将页面重定向到 `login_path` 选项定义的 URL 中。
         */
        'permission' => function () {
            // 只要是能管理内容的用户，就允许访问后台
            return Auth::check() && Auth::user()->can('manage_contents');
        },

        /*
         * 使用布尔值来设定是否使用后台主页面。
         *
         * 如值为 `true`，将使用 `dashboard_view` 定义的视图文件渲染页面；
         * 如值为 `false`，将使用 `home_page` 定义的菜单条目来作为后台主页。
         */
        'use_dashboard' => false,

        // 设置后台主页视图文件，由 `use_dashboard` 选项决定
        'dashboard_view' => '',

        // 用来作为后台主页的菜单条目，由 `use_dashboard` 选项决定，菜单指的是 `menu` 选项
        'home_page' => 'users',

        // 右上角『返回主站』按钮的链接
        'back_to_site_path' => '/',

        // 当选项 `permission` 权限检测不通过时，会重定向用户到此处设置的路径
        'login_path' => 'login',

        // 允许在登录成功后使用 Session::get('redirect') 将用户重定向到原本想要访问的后台页面
        'login_redirect_key' => 'redirect',

        // 控制模型数据列表页默认的显示条目
        'global_rows_per_page' => 20,

        // 可选的语言，如果不为空，将会在页面顶部显示『选择语言』按钮
        'locales' => [],
    );

下面着重讲下两个选项：

* `permission`
  —— 生产环境中，请谨慎定义好访问权限，否则将造成安全威胁；
* `menu`
  —— 后台管理菜单，后面新增 Model 管理时，我们将会频繁修改此选项。

### 4. 创建必要的文件夹

Administrator 会监测`settings_config_path`和`model_config_path`选项目录是否能正常访问，否则会报错，故我们先使用以下命令新建目录：

```
$ mkdir -p config/administrator/settings
$ touch config/administrator/settings/.gitkeep
```

在空文件夹中放置`.gitkeep`保证了 Git 会将此文件夹纳入版本控制器中。

## 导航入口

为了方便快速进入后台，我们新增『管理后台』导航入口，注意需要判断权限：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
            <div class="dropdown-menu" aria-labelledby="navbarDropdown">
              @can('manage_contents')
                <a class="dropdown-item" href="{{ url(config('administrator.uri')) }}">
                  <i class="fas fa-tachometer-alt mr-2"></i>
                  管理后台
                </a>
                <div class="dropdown-divider"></div>
              @endcan
              <a class="dropdown-item" href="{{ route('users.show', Auth::id()) }}">
                <i class="far fa-user mr-2"></i>
                个人中心
              </a>
.
.
.
```

刷新页面，查看顶部下拉菜单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/xcgsTBE11V.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/xcgsTBE11V.png!large)

## Git 版本控制

至此，管理后台已经集成完毕。我们需要等下一章节讲到模型配置信息后，才能测试页面显示。现在先让我们将修改添加到 Git 中：

```
$ git add -A
$ git commit -m "集成 administrator"
```



