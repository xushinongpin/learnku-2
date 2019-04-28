[![](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/gLJcJHBmAi.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/gLJcJHBmAi.png)

## 什么是多角色？

角色和权限是许多 Web 应用程序的重要组成部分。角色就是用户在站点中的身份，很多时候与站点权限相关联。

以 LaraBBS 为例，将会有以下角色，他们的权限由低到高：

* 游客 —— 没有登录的用户
* 用户 —— 登录用户
* 管理员 —— 社区内容管理
* 站长 —— 权限最高的用户角色

在我们的 LaraBBS 项目里：

* **游客**
  可以随便浏览页面，但是无法发布内容；
* **用户**
  能够发布内容，却只能管理自己的内容；
* **管理员**
  可以管理所有用户的内容，然而不能管理用户；
* **站长**
  拥有最高权限，可以管理所有内容，包括用户。

『游客』和『用户』我们只需要按照登录状态来辨别即可，『管理员』和『站长』都是登录用户，并且一个用户既可以是管理员也可以是站长。在代码中，我们使用 Role 数据模型来作为角色的表现，角色能做的动作，我们称之为权限，使用数据模型 Permission 来表现。

## 权限管理扩展包

Laravel 自带了简单的用户授权方案：

* Gates 和 Policies
* $this-
  &gt;
  authorize \(\) 方法
* @can 和 @cannot Blade 命令

不过对于我们的项目需求，自带的方案显然是不够的。幸运的是，这个领域中有很多扩展包，可以帮助我们实现 Laravel 核心功能不容易实现的权限和角色需求。本章节中，我们将使用[Laravel-permission](https://github.com/spatie/laravel-permission)，选择此扩展包有以下理由：

* 作者在积极维护；
* 详尽的文档；
* 容易理解的数据库结构；
* 跟着 Laravel 自带的授权风格走，不会产生冲突；
* 重视性能优化 —— 缓存角色和权限信息，高速读取。

以上几点，在你选择其他扩展包时，也可以作为斟酌的标准。

## 1. 安装扩展包

通过 Composer 安装：

```
$ composer require "spatie/laravel-permission:~2.29"
```

生成数据库迁移文件：

```
$ php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider" --tag="migrations"
```

下图是 laravel-permission 的数据库表结构：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/35YmbLHqjC.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/35YmbLHqjC.png)

数据表各自的作用：

* roles —— 角色的模型表；
* permissions —— 权限的模型表；
* model\_has\_roles —— 模型与角色的关联表，用户拥有什么角色在此表中定义，一个用户能拥有多个角色；
* role\_has\_permissions —— 角色拥有的权限关联表，如管理员拥有查看后台的权限都是在此表定义，一个角色能拥有多个权限；
* model\_has\_permissions —— 模型与权限关联表，一个模型能拥有多个权限。

从最后一张表中可以看出，laravel-permission 允许用户跳过角色，直接拥有权限。不过在本项目中，为了方便管理，我们设定：

> 用户只能通过角色来获取到权限，用户不单独拥有权限。例如：用户 Summer 必须是『站长』角色，才能行使『用户管理』权限。

介绍完数据表后，接下来继续使用以下命令应用数据迁移：

```
$ php artisan migrate
```

生成配置信息：

```
$ php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider" --tag="config"
```

配置信息存放于 config/permission.php ，可以打开此文件瞧一瞧，目前我们不需要做任何修改。

## 2. 加载 HasRoles

我们还需在 User 中使用 laravel-permission 提供的 Trait —— HasRoles，此举能让我们获取到扩展包提供的所有权限和角色的操作方法。

_app/Models/User.php_

```
<?php
.
.
.
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable implements MustVerifyEmailContract
{
    use HasRoles;
    .
    .
    .
}
```

## 3. 初始化角色和权限

使用角色管理之前，我们需要思考下：我们有哪些地方会用到权限管理？我们的权限和角色都有哪些？

LaraBBS 项目里，角色的话，前面我们说过了，我们会有『管理员』和『站长』。

我们允许管理员维护社区的内容，所以管理员跟一般用户的区别是能否管理站点内容，我们将这个权限定义为`manage_contents`，管理员和站长角色都拥有`manage_contents`权限。权限的命名规范我们统一只用`_`分隔法。

我们的管理员都是线上招募过来的，我们不希望给予他们太多的责任，管理员可以管理内容，但是却不能管理用户，例如修改用户密码、删除用户等。站长却需要拥有这个权限，我们将此权限定义为`manage_users`。

后面我们还会开发『站点设置』功能，站长可以在后台管理站点相关的设置，如 SEO 设置、联系邮箱等。而这个权限我们希望站长独有，我们将此权限定义为`edit_settings`。

与『话题分类』的初始化一样，我们将使用数据迁移来实现初始化角色权限相关的代码，遵照命名规范`seed_(数据库表名称)_data`：

```
$ php artisan make:migration seed_roles_and_permissions_data
```

打开迁移文件，书写初始化权限和角色的代码：

_database/migrations/{timestamp}\_seed\_roles\_and\_permissions\_data.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

use Illuminate\Database\Eloquent\Model;
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

class SeedRolesAndPermissionsData extends Migration
{
    public function up()
    {
        // 需清除缓存，否则会报错
        app(Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions();

        // 先创建权限
        Permission::create(['name' => 'manage_contents']);
        Permission::create(['name' => 'manage_users']);
        Permission::create(['name' => 'edit_settings']);

        // 创建站长角色，并赋予权限
        $founder = Role::create(['name' => 'Founder']);
        $founder->givePermissionTo('manage_contents');
        $founder->givePermissionTo('manage_users');
        $founder->givePermissionTo('edit_settings');

        // 创建管理员角色，并赋予权限
        $maintainer = Role::create(['name' => 'Maintainer']);
        $maintainer->givePermissionTo('manage_contents');
    }

    public function down()
    {
        // 需清除缓存，否则会报错
        app(Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions();

        // 清空所有数据表数据
        $tableNames = config('permission.table_names');

        Model::unguard();
        DB::table($tableNames['role_has_permissions'])->delete();
        DB::table($tableNames['model_has_roles'])->delete();
        DB::table($tableNames['model_has_permissions'])->delete();
        DB::table($tableNames['roles'])->delete();
        DB::table($tableNames['permissions'])->delete();
        Model::reguard();
    }
}
```

为了测试的方便，我们需要在生成用户填充数据以后，为 1 号和 2 号用户分别指派角色，修改`run()`方法 ：

_database/seeds/UsersTableSeeder.php_

```
<?php
.
.
.

class UsersTableSeeder extends Seeder
{
    public function run()
    {
        .
        .
        .

        // 初始化用户角色，将 1 号用户指派为『站长』
        $user->assignRole('Founder');

        // 将 2 号用户指派为『管理员』
        $user = User::find(2);
        $user->assignRole('Maintainer');
    }
}
```

`assignRole()`方法在 HasRoles 中定义，我们已在 User 模型中加载了它。

接下来刷新下数据库中的测试数据：

```
$ php artisan migrate:refresh --seed
```

## 4. 简单用法

接下来讲解下 laravel-permission 的一些简单用法。

新建角色，只需要提供`name`字段即可：

```
use Spatie\Permission\Models\Role;
$role = Role::create(['name' => 'Founder']);
```

为角色添加权限：

```
use Spatie\Permission\Models\Permission;

Permission::create(['name' => 'manage_contents']);
$role->givePermissionTo('manage_contents');
```

赋予用户某个角色：

```
// 单个角色
$user->assignRole('Founder');

// 多个角色
$user->assignRole('writer', 'admin');

// 数组形式的多个角色
$user->assignRole(['writer', 'admin']);
```

我们可以使用以下方法来检查用户角色：

```
// 是否是站长
$user->hasRole('Founder');

// 是否拥有至少一个角色
$user->hasAnyRole(Role::all());  

// 是否拥有所有角色
$user->hasAllRoles(Role::all());   
```

检查权限：

```
// 检查用户是否有某个权限
$user->can('manage_contents'); 

// 检查角色是否拥有某个权限
$role->hasPermissionTo('manage_contents');  
```

直接给用户添加权限：

```
// 为用户添加『直接权限』
$user->givePermissionTo('manage_contents');

// 获取所有直接权限
$user->getDirectPermissions() 
```

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "集成 laravel-permission 扩展包"
```



