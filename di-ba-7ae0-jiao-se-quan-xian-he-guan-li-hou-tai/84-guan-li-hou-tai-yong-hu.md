## 模型配置信息

Administrator 的运行机制是通过解析『模型配置信息』来生成后台，每一个模型配置文件对应一个数据模型，同时也对应一个页面。目前我们的项目里有 6 个数据模型，接下来我们将分别为他们定制模型配置信息。

## 配置信息与后台布局

配置信息主要由三个布局选项，再加上其他如数据模型、权限控制、表单验证规则等选项构成。开始之前我们先讲解下三个布局在页面上的位置，以此来帮助我们更好地理解『模型配置信息』选项。

### 布局选项

三个布局选项分别是：

* 1\). 数据表格 —— 对应选项
  `columns`
  ，用来列表数据，支持分页和批量删除；
* 2\). 模型表单 —— 对应选项
  `edit_fields`
  ，用来新建和编辑模型数据；
* 3\). 数据过滤 —— 对应选项
  `filters`
  ，与数据表格实时响应的表单，用来筛选数据。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/bEEf8zImKN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/bEEf8zImKN.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Q6irsKKPUw.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Q6irsKKPUw.png!large)

### 必填选项

除了上面提到的三个布局选项，一个合格模型配置信息里，还必须包含以下选项：

* title —— 标题设置
* single —— 模型单数，用作新建按钮的命名如『新建 $single』
* model —— 数据模型，用作数据的 CRUD

## 用户模型配置 users

接下来开始我们第一个数据模型的配置，请注意看代码注释：

_config/administrator/users.php_

```
<?php

use App\Models\User;

return [
    // 页面标题
    'title'   => '用户',

    // 模型单数，用作页面『新建 $single』
    'single'  => '用户',

    // 数据模型，用作数据的 CRUD
    'model'   => User::class,

    // 设置当前页面的访问权限，通过返回布尔值来控制权限。
    // 返回 True 即通过权限验证，False 则无权访问并从 Menu 中隐藏
    'permission'=> function()
    {
        return Auth::user()->can('manage_users');
    },

    // 字段负责渲染『数据表格』，由无数的『列』组成，
    'columns' => [

        // 列的标示，这是一个最小化『列』信息配置的例子，读取的是模型里对应
        // 的属性的值，如 $model->id
        'id',

        'avatar' => [
            // 数据表格里列的名称，默认会使用『列标识』
            'title'  => '头像',

            // 默认情况下会直接输出数据，你也可以使用 output 选项来定制输出内容
            'output' => function ($avatar, $model) {
                return empty($avatar) ? 'N/A' : '<img src="'.$avatar.'" width="40">';
            },

            // 是否允许排序
            'sortable' => false,
        ],

        'name' => [
            'title'    => '用户名',
            'sortable' => false,
            'output' => function ($name, $model) {
                return '<a href="/users/'.$model->id.'" target=_blank>'.$name.'</a>';
            },
        ],

        'email' => [
            'title' => '邮箱',
        ],

        'operation' => [
            'title'  => '管理',
            'sortable' => false,
        ],
    ],

    // 『模型表单』设置项
    'edit_fields' => [
        'name' => [
            'title' => '用户名',
        ],
        'email' => [
            'title' => '邮箱',
        ],
        'password' => [
            'title' => '密码',

            // 表单使用 input 类型 password
            'type' => 'password',
        ],
        'avatar' => [
            'title' => '用户头像',

            // 设置表单条目的类型，默认的 type 是 input
            'type' => 'image',

            // 图片上传必须设置图片存放路径
            'location' => public_path() . '/uploads/images/avatars/',
        ],
        'roles' => [
            'title'      => '用户角色',

            // 指定数据的类型为关联模型
            'type'       => 'relationship',

            // 关联模型的字段，用来做关联显示
            'name_field' => 'name',
        ],
    ],

    // 『数据过滤』设置
    'filters' => [
        'id' => [

            // 过滤表单条目显示名称
            'title' => '用户 ID',
        ],
        'name' => [
            'title' => '用户名',
        ],
        'email' => [
            'title' => '邮箱',
        ],
    ],
];
```

请确保使用 ID 为 1 的用户，这是我们的管理员用户，然后导航栏下拉菜单选择『管理后台』即可进入：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/4pfMAo4SlB.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/4pfMAo4SlB.png!large)

## 1\). 测试数据筛选

尝试一下实时响应的数据筛选：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/jWJ2lldQEL.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/jWJ2lldQEL.gif!large)

## 2\). 测试模型编辑

测试模型编辑功能：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Nl0oALf6tX.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Nl0oALf6tX.gif!large)

## 3\). 测试删除功能

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CgFT5Bh25E.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CgFT5Bh25E.gif!large)

## 4\). 修改密码

为了方便测试，我们复制邮箱来作为第 9 号用户的密码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/AadefAfOiN.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/AadefAfOiN.gif!large)

返回主页，退出登录，然后重新进入登录页面，我们使用 9 号用户进行登录，账号和密码都是邮箱：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/yl9CdGlk5p.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/yl9CdGlk5p.gif!large)

我们的修改密码存在问题，接下来看看如何解决。

我们需要先定位问题。可以先看看数据库`users`表里的数据，瞧一瞧我们修改后的最终数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/xDY6F7eZ35.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/xDY6F7eZ35.png!large)

Laravel 的登录机制是使用了[password\_verify](http://php.net/manual/zh/function.password-verify.php)函数进行匹配验证，此函数接受两个参数 $password 和 $hash。在 Laravel 中，默认的登录功能这两个参数的值一般为：

* $password —— 是登录表单提交的明文密码；
* $hash —— 是哈希加密以后的数据，存储在数据表
  `users`
  表里的
  `password`
  字段。

所以导致我们无法登录的原因是存入了未加密的密码。解决方法是在存入数据库之前对数据做下加密即可。

Administrator 并未提供给我们修改用户提交数据的方法，不过这种针对模型属性的操作，Eloquent 已经为我们提供了非常便捷的方法 ——[Eloquent 修改器](https://learnku.com/docs/laravel/5.7/eloquent-mutators)。

在 Eloquent 模型实例中获取或设置某些属性值的时候，访问器和修改器允许你对 Eloquent 属性值进行格式化。有两种方法可以修改 Eloquent 模型属性的值，一种是『访问器』，另一种是『修改器』。

访问器和修改器最大的区别是『发生修改的时机』，访问器是**访问属性时**修改，修改器是在**写入数据库前**修改。修改器是数据持久化，访问器是临时修改。访问器的使用场景是当数据因为特殊原因存在不一致性时，可以使用访问器进行矫正处理。在我们的密码加密的场景里，我们会使用修改器在密码即将入库前，对其进行加密。

我们须在模型上定义一个`setPasswordAttribute`方法，注意命名规范是`set{属性的驼峰式命名}Attribute`，当我们给属性赋值时，如`$user->password = 'password'`，该修改器将被自动调用：

```
public function setPasswordAttribute($value)
{
    $this->attributes['password'] = bcrypt($value);
}
```

需要注意的是，我们还需考虑项目中是否其他地方也会修改到`password`字段？答案是有的，Laravel 自带的用户登录注册功能里，也有密码重置功能，我们来阅读下其源代码。

ResetPasswordController 将重设密码的逻辑写在`ResetsPasswords`trait 里，打开这个文件，定位到密码更改并入库发生的方法`resetPassword`：

_vendor/laravel/framework/src/Illuminate/Foundation/Auth/ResetsPasswords.php_

```
protected function resetPassword($user, $password)
{
    $user->password = Hash::make($password);
    .
    .
    .
}
```

可以看到密码是经过哈希以后才赋值的，所以当我们在修改器中接收到数据时，应该会加密过的和未加密的数据。我们对这两种情况做区分对待：

_app/Models/User.php_

```
<?php
.
.
.
class User extends Authenticatable implements MustVerifyEmailContract
{
    .
    .
    .

    public function setPasswordAttribute($value)
    {
        // 如果值的长度等于 60，即认为是已经做过加密的情况
        if (strlen($value) != 60) {

            // 不等于 60，做密码加密处理
            $value = bcrypt($value);
        }

        $this->attributes['password'] = $value;
    }
}
```

现在按照上面的测试步骤再次尝试，即可顺利登录 9 号用户：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CdI7n74swA.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/CdI7n74swA.gif!large)

## 5\). 修改用户头像

接下来我们测试刚刚用户头像。使用 1 号用户 Summer 登录后台，并尝试对 9 号用户进行头像修改：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/NNP9kJBSSp.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/NNP9kJBSSp.gif!large)

会发现修改完成后的图片无法显示。还是使用老方法来调试，先看下数据库到底存了什么数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/qf5euNnXSp.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/qf5euNnXSp.png!large)

原来是因为链接不合法，只是一个文件名，当然无法显示图片。接下来我们将同样使用修改器来修复此问题，为即将入库的数据拼接完整的 URL 。同时，我们也需要考虑到其他地方对`avatar`的赋值，用户修改资料时，我们为`avatar`的赋值就是全路径，因此同样的我们需要考虑两种状况：

_app/Models/User.php_

    <?php
    .
    .
    .
    class User extends Authenticatable implements MustVerifyEmailContract
    {
        .
        .
        .

        public function setAvatarAttribute($path)
        {
            // 如果不是 `http` 子串开头，那就是从后台上传的，需要补全 URL
            if ( ! starts_with($path, 'http')) {

                // 拼接完整的 URL
                $path = config('app.url') . "/uploads/images/avatars/$path";
            }

            $this->attributes['avatar'] = $path;
        }
    }

再次尝试，即可成功上传图片：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/hfiBlQ1iRS.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/hfiBlQ1iRS.gif!large)

## Git 版本控制

将修改添加到 Git 中：

```
$ git add -A
$ git commit -m "用户管理后台 "
```



