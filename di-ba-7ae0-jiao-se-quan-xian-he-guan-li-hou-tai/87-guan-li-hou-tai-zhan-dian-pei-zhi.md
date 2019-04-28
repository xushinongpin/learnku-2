## 站点配置

Administrator 自带了站点设置功能，接下来我们利用此功能，允许管理员通过后台设置，来配置站点 SEO 信息、更改联系人邮件等。

站点设置的开发需要涉及到以下：

* 后台设置站点配置信息
* 代码中使用配置信息

## 后台设置站点配置信息

### 1. 修改 Administrator 配置信息

修改`menu`选项，新增『站点管理』子菜单：

_config/administrator.php_

```
<?php

return array(
.
.
.
    'menu' => [
        '用户与权限' => [
            'users',
            'roles',
            'permissions',
        ],
        '内容管理' => [
            'categories',
            'topics',
            'replies',
        ],
        '站点管理' => [
            'settings.site',
        ],
    ],
.
.
.
);
```

> 需要注意的是此类后台，需要在菜单里使用`settings.`前缀，并且将文件放置于`settings_config_path`定义的目录中。

### 2. 新增配置信息

接下来是新增配置信息：

_config/administrator/settings/site.php_

```
<?php

return [
    'title' => '站点设置',

    // 访问权限判断
    'permission'=> function()
    {
        // 只允许站长管理站点配置
        return Auth::user()->hasRole('Founder');
    },

    // 站点配置的表单
    'edit_fields' => [
        'site_name' => [
            // 表单标题
            'title' => '站点名称',

            // 表单条目类型
            'type' => 'text',

            // 字数限制
            'limit' => 50,
        ],
        'contact_email' => [
            'title' => '联系人邮箱',
            'type' => 'text',
            'limit' => 50,
        ],
        'seo_description' => [
            'title' => 'SEO - Description',
            'type' => 'textarea',
            'limit' => 250,
        ],
        'seo_keyword' => [
            'title' => 'SEO - Keywords',
            'type' => 'textarea',
            'limit' => 250,
        ],
    ],

    // 表单验证规则
    'rules' => [
        'site_name' => 'required|max:50',
        'contact_email' => 'email',
    ],

    'messages' => [
        'site_name.required' => '请填写站点名称。',
        'contact_email.email' => '请填写正确的联系人邮箱格式。',
    ],

    // 数据即将保持的触发的钩子，可以对用户提交的数据做修改
    'before_save' => function(&$data)
    {
        // 为网站名称加上后缀，加上判断是为了防止多次添加
        if (strpos($data['site_name'], 'Powered by LaraBBS') === false) {
            $data['site_name'] .= ' - Powered by LaraBBS';
        }
    },

    // 你可以自定义多个动作，每一个动作为设置页面底部的『其他操作』区块
    'actions' => [

        // 清空缓存
        'clear_cache' => [
            'title' => '更新系统缓存',

            // 不同状态时页面的提醒
            'messages' => [
                'active' => '正在清空缓存...',
                'success' => '缓存已清空！',
                'error' => '清空缓存时出错！',
            ],

            // 动作执行代码，注意你可以通过修改 $data 参数更改配置信息
            'action' => function(&$data)
            {
                \Artisan::call('cache:clear');
                return true;
            }
        ],
    ],
];
```

### 3. 开始测试

通过顶部导航栏进入站点设置页面，表单里填入以下内容并保存：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/I4uhVNfY2I.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/I4uhVNfY2I.png!large)

## 代码中使用配置信息

Administrator 自带了一个辅助函数`setting()`允许我们来获取设置信息：

```
setting($key, $default = '', $setting_name = 'site');
```

下面是获取『站点名称』的例子：

```
$site_name = setting('site_name');
```

此方法允许传入默认值：

```
$site_name = setting('site_name', '默认站点名称');
```

### 1. 联系人邮件

_resources/views/layouts/\_footer.blade.php_

```
<footer class="footer">
  <div class="container">
    .
    .
    .

    <p class="float-right"><a href="mailto:{{ setting('contact_email') }}">联系我们</a></p>
  </div>
</footer>
```

查看页面源码，即可看到我们后台设定的邮箱地址：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Uq18rkUh4B.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Uq18rkUh4B.gif!large)

### 2. 页面 SEO 信息

_resources/views/layouts/app.blade.php_

```
.
.
.
  <!-- CSRF Token -->
  <meta name="csrf-token" content="{{ csrf_token() }}">

  <title>@yield('title', 'LaraBBS') - {{ setting('site_name', 'Laravel 进阶教程') }}</title>
  <meta name="description" content="@yield('description', setting('seo_description', 'LaraBBS 爱好者社区。'))" />
  <meta name="keyword" content="@yield('keyword', setting('seo_keyword', 'LaraBBS,社区,论坛,开发者论坛'))" />

  <!-- Styles -->
.
.
.
```

刷新页面并查看页面源代码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/hbDITd2gS1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/hbDITd2gS1.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "站点设置"
```



