## 内容管理

本章节我们将配置内容管理后台，分别是`categories`,`topics`和`replies`。

## 1. 修改 Administrator 配置信息

修改`menu`选项，新增内容管理子菜单：

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
    ],
.
.
.
);
```

## 2. 分类模型配置

新建分类模型配置文件，请仔细阅读代码注释：

_config/administrator/categories.php_

```
<?php

use App\Models\Category;

return [
    'title'   => '分类',
    'single'  => '分类',
    'model'   => Category::class,

    // 对 CRUD 动作的单独权限控制，其他动作不指定默认为通过
    'action_permissions' => [
        // 删除权限控制
        'delete' => function () {
            // 只有站长才能删除话题分类
            return Auth::user()->hasRole('Founder');
        },
    ],

    'columns' => [
        'id' => [
            'title' => 'ID',
        ],
        'name' => [
            'title'    => '名称',
            'sortable' => false,
        ],
        'description' => [
            'title'    => '描述',
            'sortable' => false,
        ],
        'operation' => [
            'title'  => '管理',
            'sortable' => false,
        ],
    ],
    'edit_fields' => [
        'name' => [
            'title' => '名称',
        ],
        'description' => [
            'title' => '描述',
            'type'  => 'textarea',
        ],
    ],
    'filters' => [
        'id' => [
            'title' => '分类 ID',
        ],
        'name' => [
            'title' => '名称',
        ],
        'description' => [
            'title' => '描述',
        ],
    ],
    'rules'   => [
        'name' => 'required|min:1|unique:categories'
    ],
    'messages' => [
        'name.unique'   => '分类名在数据库里有重复，请选用其他名称。',
        'name.required' => '请确保名字至少一个字符以上',
    ],
];
```

## 3. 话题模型配置

_config/administrator/topics.php_

```
<?php

use App\Models\Topic;

return [
    'title'   => '话题',
    'single'  => '话题',
    'model'   => Topic::class,

    'columns' => [

        'id' => [
            'title' => 'ID',
        ],
        'title' => [
            'title'    => '话题',
            'sortable' => false,
            'output'   => function ($value, $model) {
                return '<div style="max-width:260px">' . model_link($value, $model) . '</div>';
            },
        ],
        'user' => [
            'title'    => '作者',
            'sortable' => false,
            'output'   => function ($value, $model) {
                $avatar = $model->user->avatar;
                $value = empty($avatar) ? 'N/A' : '<img src="'.$avatar.'" style="height:22px;width:22px"> ' . $model->user->name;
                return model_link($value, $model->user);
            },
        ],
        'category' => [
            'title'    => '分类',
            'sortable' => false,
            'output'   => function ($value, $model) {
                return model_admin_link($model->category->name, $model->category);
            },
        ],
        'reply_count' => [
            'title'    => '评论',
        ],
        'operation' => [
            'title'  => '管理',
            'sortable' => false,
        ],
    ],
    'edit_fields' => [
        'title' => [
            'title'    => '标题',
        ],
        'user' => [
            'title'              => '用户',
            'type'               => 'relationship',
            'name_field'         => 'name',

            // 自动补全，对于大数据量的对应关系，推荐开启自动补全，
            // 可防止一次性加载对系统造成负担
            'autocomplete'       => true,

            // 自动补全的搜索字段
            'search_fields'      => ["CONCAT(id, ' ', name)"],

            // 自动补全排序
            'options_sort_field' => 'id',
        ],
        'category' => [
            'title'              => '分类',
            'type'               => 'relationship',
            'name_field'         => 'name',
            'search_fields'      => ["CONCAT(id, ' ', name)"],
            'options_sort_field' => 'id',
        ],
        'reply_count' => [
            'title'    => '评论',
        ],
        'view_count' => [
            'title'    => '查看',
        ],
    ],
    'filters' => [
        'id' => [
            'title' => '内容 ID',
        ],
        'user' => [
            'title'              => '用户',
            'type'               => 'relationship',
            'name_field'         => 'name',
            'autocomplete'       => true,
            'search_fields'      => array("CONCAT(id, ' ', name)"),
            'options_sort_field' => 'id',
        ],
        'category' => [
            'title'              => '分类',
            'type'               => 'relationship',
            'name_field'         => 'name',
            'search_fields'      => array("CONCAT(id, ' ', name)"),
            'options_sort_field' => 'id',
        ],
    ],
    'rules'   => [
        'title' => 'required'
    ],
    'messages' => [
        'title.required' => '请填写标题',
    ],
];
```

出现了`model_link`和`model_admin_link`的函数，我们需要在`helpers.php`中定义：

_app/helpers.php_

    <?php
    .
    .
    .

    function model_admin_link($title, $model)
    {
        return model_link($title, $model, 'admin');
    }

    function model_link($title, $model, $prefix = '')
    {
        // 获取数据模型的复数蛇形命名
        $model_name = model_plural_name($model);

        // 初始化前缀
        $prefix = $prefix ? "/$prefix/" : '/';

        // 使用站点 URL 拼接全量 URL
        $url = config('app.url') . $prefix . $model_name . '/' . $model->id;

        // 拼接 HTML A 标签，并返回
        return '<a href="' . $url . '" target="_blank">' . $title . '</a>';
    }

    function model_plural_name($model)
    {
        // 从实体中获取完整类名，例如：App\Models\User
        $full_class_name = get_class($model);

        // 获取基础类名，例如：传参 `App\Models\User` 会得到 `User`
        $class_name = class_basename($full_class_name);

        // 蛇形命名，例如：传参 `User`  会得到 `user`, `FooBar` 会得到 `foo_bar`
        $snake_case_name = snake_case($class_name);

        // 获取子串的复数形式，例如：传参 `user` 会得到 `users`
        return str_plural($snake_case_name);
    }

## 4. 回复模型配置

_config/administrator/replies.php_

```
<?php

use App\Models\Reply;

return [
    'title'   => '回复',
    'single'  => '回复',
    'model'   => Reply::class,

    'columns' => [

        'id' => [
            'title' => 'ID',
        ],
        'content' => [
            'title'    => '内容',
            'sortable' => false,
            'output'   => function ($value, $model) {
                return '<div style="max-width:220px">' . $value . '</div>';
            },
        ],
        'user' => [
            'title'    => '作者',
            'sortable' => false,
            'output'   => function ($value, $model) {
                $avatar = $model->user->avatar;
                $value = empty($avatar) ? 'N/A' : '<img src="'.$avatar.'" style="height:22px;width:22px"> ' . $model->user->name;
                return model_link($value, $model->user);
            },
        ],
        'topic' => [
            'title'    => '话题',
            'sortable' => false,
            'output'   => function ($value, $model) {
                return '<div style="max-width:260px">' . model_admin_link($model->topic->title, $model->topic) . '</div>';
            },
        ],
        'operation' => [
            'title'  => '管理',
            'sortable' => false,
        ],
    ],
    'edit_fields' => [
        'user' => [
            'title'              => '用户',
            'type'               => 'relationship',
            'name_field'         => 'name',
            'autocomplete'       => true,
            'search_fields'      => array("CONCAT(id, ' ', name)"),
            'options_sort_field' => 'id',
        ],
        'topic' => [
            'title'              => '话题',
            'type'               => 'relationship',
            'name_field'         => 'title',
            'autocomplete'       => true,
            'search_fields'      => array("CONCAT(id, ' ', title)"),
            'options_sort_field' => 'id',
        ],
        'content' => [
            'title'    => '回复内容',
            'type'     => 'textarea',
        ],
    ],
    'filters' => [
        'user' => [
            'title'              => '用户',
            'type'               => 'relationship',
            'name_field'         => 'name',
            'autocomplete'       => true,
            'search_fields'      => array("CONCAT(id, ' ', name)"),
            'options_sort_field' => 'id',
        ],
        'topic' => [
            'title'              => '话题',
            'type'               => 'relationship',
            'name_field'         => 'title',
            'autocomplete'       => true,
            'search_fields'      => array("CONCAT(id, ' ', title)"),
            'options_sort_field' => 'id',
        ],
        'content' => [
            'title'    => '回复内容',
        ],
    ],
    'rules'   => [
        'content' => 'required'
    ],
    'messages' => [
        'content.required' => '请填写回复内容',
    ],
];
```

## 5. 测试一下

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/HuWW4k4uuo.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/HuWW4k4uuo.gif!large)

这是数据损坏所致 —— 我们删除了用户，却没有删除用户发布的话题，此部分话题变成了遗留数据。话题列表中渲染到这些遗留数据时，因为不存在作者，却取作者的`avatar`头像属性，故报错：

> Trying to get property of non-object

因为话题和分类下面分别关联着回复和话题，如果你测试时删除话题和分类后去查看回复列表和话题列表，也可能会因为这个问题报错。

这个问题我们将会在后面的章节中解决，这里我们重新运行 Artisan 命令行重新生成干净的数据即可：

```
$ php artisan migrate:refresh --seed
```

每一次运行此命令，都能体会到 Laravel 开发的便捷性，测试数据一个命令就生成了。

## XSS 安全须知

默认情况下，Administrator 会对所有的输出内容进行`htmlspecialchars()`过滤，以此来防范 XSS 攻击。然而当你在『数据表格』中使用`output`选项输出 HTML 时，Administrator 将无法保护你：

需要谨记的一点 ——『永远不要相信用户能够修改的数据』，`output`选项接受的两个参数里，`$value`的数据是被过滤过的，可以安全使用。其他的用户数据，输出时请使用 Laravel 自带的`e()`函数对其做转义处理，如下面的例子：

```
'topic' => [
    'title'    => '话题',
    'sortable' => false,
    'output'   => function ($value, $model) {
        return '<div style="max-width:260px">' . model_admin_link(e($model->topic->title), $model->topic) . '</div>';
    },
],
```

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "内容管理后台"
```



