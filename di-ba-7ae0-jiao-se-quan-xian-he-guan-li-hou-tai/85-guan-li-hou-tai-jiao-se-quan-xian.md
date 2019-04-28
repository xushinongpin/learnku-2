## 角色权限管理后台

本章节我们将配置角色权限管理后台，分别是`roles`和`permissions`。

## 1. 修改 Administrator 配置信息

修改`menu`选项，新增`roles`和`permissions`：

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
    ],
.
.
.
);
```

## 2. 新建模型配置

新建角色配置文件，请仔细阅读代码注释：

_config/administrator/roles.php_

```
<?php

use Spatie\Permission\Models\Role;

return [
    'title'   => '角色',
    'single'  => '角色',
    'model'   => Role::class,

    'permission'=> function()
    {
        return Auth::user()->can('manage_users');
    },

    'columns' => [
        'id' => [
            'title' => 'ID',
        ],
        'name' => [
            'title' => '标识'
        ],
        'permissions' => [
            'title'  => '权限',
            'output' => function ($value, $model) {
                $model->load('permissions');
                $result = [];
                foreach ($model->permissions as $permission) {
                    $result[] = $permission->name;
                }

                return empty($result) ? 'N/A' : implode($result, ' | ');
            },
            'sortable' => false,
        ],
        'operation' => [
            'title'  => '管理',
            'output' => function ($value, $model) {
                return $value;
            },
            'sortable' => false,
        ],
    ],

    'edit_fields' => [
        'name' => [
            'title' => '标识',
        ],
        'permissions' => [
            'type' => 'relationship',
            'title' => '权限',
            'name_field' => 'name',
        ],
    ],

    'filters' => [
        'id' => [
            'title' => 'ID',
        ],
        'name' => [
            'title' => '标识',
        ]
    ],

    // 新建和编辑时的表单验证规则
    'rules' => [
        'name' => 'required|max:15|unique:roles,name',
    ],

    // 表单验证错误时定制错误消息
    'messages' => [
        'name.required' => '标识不能为空',
        'name.unique' => '标识已存在',
    ]
];
```

文件尾部新增了两个选项 ——`rules`和`messages`，用来验证『模型表单』的提交数据。

接下来新建权限控制的配置信息：

_config/administrator/permissions.php_

```
<?php

use Spatie\Permission\Models\Permission;

return [
    'title'   => '权限',
    'single'  => '权限',
    'model'   => Permission::class,

    'permission' => function () {
        return Auth::user()->can('manage_users');
    },

    // 对 CRUD 动作的单独权限控制，通过返回布尔值来控制权限。
    'action_permissions' => [
        // 控制『新建按钮』的显示
        'create' => function ($model) {
            return true;
        },
        // 允许更新
        'update' => function ($model) {
            return true;
        },
        // 不允许删除
        'delete' => function ($model) {
            return false;
        },
        // 允许查看
        'view' => function ($model) {
            return true;
        },
    ],

    'columns' => [
        'id' => [
            'title' => 'ID',
        ],
        'name' => [
            'title'    => '标示',
        ],
        'operation' => [
            'title'    => '管理',
            'sortable' => false,
        ],
    ],

    'edit_fields' => [
        'name' => [
            'title' => '标示（请慎重修改）',

            // 表单条目标题旁的『提示信息』
            'hint' => '修改权限标识会影响代码的调用，请不要轻易更改。'
        ],
        'roles' => [
            'type' => 'relationship',
            'title' => '角色',
            'name_field' => 'name',
        ],
    ],

    'filters' => [
        'name' => [
            'title' => '标示',
        ],
    ],
];
```

新增了`action_permissions`选项，Administrator 允许我们对 CRUD 动作做单独的权限控制，因为角色权限是写死在代码中的，所以此处我们禁止管理员执行删除操作，以防止造成不必要的混乱。

## 3. 开始测试

查看下数据，点击编辑按钮，查看互相关联的角色和权限：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/0gN8WIF1MV.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/0gN8WIF1MV.gif!large)

可以尝试修改下权限和角色的关联，不过为了后面课程进展顺利，请修改回来。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "管理员可以管理权限和角色"
```



