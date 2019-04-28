## 假数据填充

目前我们数据库中的帖子数据为空，因此[话题列表页面](http://larabbs.test/topics)如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/IdUotF8kHF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/IdUotF8kHF.png!large)

在开始开发话题列表之前，我们需要一些假数据来辅助，假数据生成逻辑如下：

* 填充 10 条用户数据，作为话题的作者使用；
* 100 条话题数据，这样我们就能测试分页功能；
* 填充话题时分类随机；
* 填充话题时作者随机。

## 一、填充用户数据

话题数据中需使用『用户数据』作为话题作者，有依赖关系，故我们先填充用户数据。

用户的假数据填充涉及到以下几个文件：

1. 数据模型 User.php
2. 用户的数据工厂 database/factories/UserFactory.php
3. 用户的数据填充 database/seeds/UsersTableSeeder.php
4. 注册数据填充 database/seeds/DatabaseSeeder.php

数据模型在前面章节中已定制过，此处无需修改，接下来我们从 UserFactory 开始。

### 1. 用户的数据工厂

Laravel 框架自带了 UserFactory.php 作为示例文件：

_database/factories/UserFactory.php_

```
<?php

use Faker\Generator as Faker;

$factory->define(App\Models\User::class, function (Faker $faker) {
    return [
        'name' => $faker->name,
        'email' => $faker->unique()->safeEmail,
        'email_verified_at' => now(),
        'password' => '$2y$10$TKh8H1.PfQx37YgCzwiKb.KjNyWgaHb9cbcoQgdIVFlYg7B77UdFm', // secret
        'remember_token' => str_random(10),
    ];
});
```

`define`定义了一个指定数据模型（如此例子`User`）的模型工厂。`define`方法接收两个参数，第一个参数为指定的 Eloquent 模型类，第二个参数为一个闭包函数，该闭包函数接收一个`Faker`PHP 函数库的实例，让我们可以在函数内部使用 Faker 方法来生成假数据并为模型的指定字段赋值。

我们需要增加`introduction`用户简介字段的填充，另外我们计划在 UsersTableSeeder 里使用[批量入库](https://learnku.com/courses/laravel-specification/516/data-filling)的方式填充数据，因此需要自行填充`created_at`和`updated_at`两个字段。修改后的代码如下：

_database/factories/UserFactory.php_

```
<?php

use Faker\Generator as Faker;

$factory->define(App\Models\User::class, function (Faker $faker) {
    $date_time = $faker->date . ' ' . $faker->time;
    return [
        'name' => $faker->name,
        'email' => $faker->unique()->safeEmail,
        'email_verified_at' => now(),
        'password' => '$2y$10$TKh8H1.PfQx37YgCzwiKb.KjNyWgaHb9cbcoQgdIVFlYg7B77UdFm', // secret
        'introduction' => $faker->sentence(),
        'created_at' => $date_time,
        'updated_at' => $date_time,
    ];
});
```

[Faker](https://github.com/fzaninotto/Faker)是一个假数据生成库，`sentence()`是 faker 提供的 API ，随机生成『小段落』文本。我们用来填充`introduction`个人简介字段。

### 2. 用户数据填充

使用以下命令生成数据填充文件：

```
$ php artisan make:seed UsersTableSeeder
```

修改文件如以下：

_database/seeds/UsersTableSeeder.php_

```
<?php

use Illuminate\Database\Seeder;
use App\Models\User;

class UsersTableSeeder extends Seeder
{
    public function run()
    {
        // 获取 Faker 实例
        $faker = app(Faker\Generator::class);

        // 头像假数据
        $avatars = [
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/s5ehp11z6s.png',
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/Lhd1SHqu86.png',
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/LOnMrqbHJn.png',
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/xAuDMxteQy.png',
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/ZqM7iaP4CR.png',
            'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/NDnzMutoxX.png',
        ];

        // 生成数据集合
        $users = factory(User::class)
                        ->times(10)
                        ->make()
                        ->each(function ($user, $index)
                            use ($faker, $avatars)
        {
            // 从头像数组中随机取出一个并赋值
            $user->avatar = $faker->randomElement($avatars);
        });

        // 让隐藏字段可见，并将数据集合转换为数组
        $user_array = $users->makeVisible(['password', 'remember_token'])->toArray();

        // 插入到数据库中
        User::insert($user_array);

        // 单独处理第一个用户的数据
        $user = User::find(1);
        $user->name = 'Summer';
        $user->email = 'summer@example.com';
        $user->avatar = 'https://iocaffcdn.phphub.org/uploads/images/201710/14/1/ZqM7iaP4CR.png';
        $user->save();

    }
}
```

#### 代码讲解

顶部使用`use`关键词引入必要的类。

`factory(User::class)`根据指定的`User`生成模型工厂构造器，对应加载`UserFactory.php`中的工厂设置。

`times(10)`指定生成模型的数量，此处我们只需要生成 10 个用户数据。

`make()`方法会将结果生成为[集合对象](https://learnku.com/docs/laravel/5.7/collections)。

`each()`是[集合对象](https://learnku.com/docs/laravel/5.7/collections)提供的[方法](https://learnku.com/docs/laravel/5.7/collections#method-each)，用来迭代集合中的内容并将其传递到回调函数中。

`use`是 PHP 中匿名函数提供的本地变量传递机制，匿名函数中必须通过`use`声明的引用，才能使用本地变量。

`makeVisible()`是 Eloquent 对象提供的方法，可以显示 User 模型`$hidden`属性里指定隐藏的字段，此操作确保入库时数据库不会报错。

### 3. 注册数据填充

接下来注册数据填充类，先去掉`$this->call(UsersTableSeeder::class);`的注释，还要注释掉我们还未开发 TopicsTableSeeder 调用：

_database/seeds/DatabaseSeeder.php_

```
<?php

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        $this->call(UsersTableSeeder::class);
        // $this->call(TopicsTableSeeder::class);
    }
}
```

### 4. 测试一下

使用以下命令运行迁移文件：

```
$ php artisan db:seed
```

查看数据库：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/pYJZrFxpKL.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/pYJZrFxpKL.png!large)

数一下发现有 11 个用户，你的应该和我不一样，因为我们之前开发注册了一些用户。我们并不想要这些数据。我们需要能删除`users`表数据，并重新生成填充数据的命令：

```
$ php artisan migrate:refresh --seed
```

Laravel 的`migrate:refresh`命令会回滚数据库的所有迁移，并运行`migrate`命令，`--seed`选项会同时运行`db:seed`命令。

执行以后可以看到我们想要的效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/ZeNMB8BPQN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/ZeNMB8BPQN.png!large)

> 注：开发时尽量不要手动往数据库里写入内容，因为类似于`migrate:refresh`这种操作是很频繁的，如果你想要数据库里有一些数据，请使用数据填充功能。这样做除了能被纳入版本控制以外，另一个好处是能让你不需要依赖数据库里的数据，这在团队协作中尤其重要，因为队友很多时候不是和你使用同一个数据库。希望同学们尽早养成好习惯。

## 二、填充话题数据

用户数据已经准备好，接下来我们处理话题数据填充。话题的假数据填充涉及到以下几个文件：

1. 数据模型 Topic.php
2. 话题的数据工厂 database/factories/TopicFactory.php
3. 话题的数据填充 database/seeds/TopicsTableSeeder.php
4. 注册数据填充 database/seeds/DatabaseSeeder.php

代码生成器已经为我们生成了 1、2、3 所需要的文件，并且自动在 DatabaseSeeder 中加入了 TopicsTableSeeder 的调用。

### 1. 数据模型

接下来我们先看看数据模型文件：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
    protected $fillable = ['title', 'body', 'user_id', 'category_id', 'reply_count', 'view_count', 'last_reply_user_id', 'order', 'excerpt', 'slug'];
}
```

可以看到生成器已经为我们自动写入`$fillable`允许填充字段数组，这里我们先不做更改。

### 2. 话题的数据工厂

接下来看`TopicFactory`，此文件定义了每一份`Topic`假数据的生成规则：

_database/factories/TopicFactory.php_

```
<?php

use Faker\Generator as Faker;

$factory->define(App\Models\Topic::class, function (Faker $faker) {
    return [
        // 'name' => $faker->name,
    ];
});
```

让我们对生成的模型工厂文件进行修改，修改后如下：

_database/factories/TopicFactory.php_

```
<?php

use Faker\Generator as Faker;

$factory->define(App\Models\Topic::class, function (Faker $faker) {

    $sentence = $faker->sentence();

    // 随机取一个月以内的时间
    $updated_at = $faker->dateTimeThisMonth();

    // 传参为生成最大时间不超过，因为创建时间需永远比更改时间要早
    $created_at = $faker->dateTimeThisMonth($updated_at);

    return [
        'title' => $sentence,
        'body' => $faker->text(),
        'excerpt' => $sentence,
        'created_at' => $created_at,
        'updated_at' => $updated_at,
    ];
});
```

我们使用`sentence()`随机生成『小段落』文本用以填充`title`标题字段和`excerpt`摘录字段。`text()`方法会生成大段的随机文本，此处我们来填充话题的`body`内容字段。两个时间的填充逻辑请仔细阅读代码注释。

### 3. 话题的数据填充

完成 TopicFactory 数据工厂的定制后，接下来我们开始书写填充部分的逻辑：

_database/seeds/TopicsTableSeeder.php_

```
<?php

use Illuminate\Database\Seeder;
use App\Models\Topic;
use App\Models\User;
use App\Models\Category;

class TopicsTableSeeder extends Seeder
{
    public function run()
    {
        // 所有用户 ID 数组，如：[1,2,3,4]
        $user_ids = User::all()->pluck('id')->toArray();

        // 所有分类 ID 数组，如：[1,2,3,4]
        $category_ids = Category::all()->pluck('id')->toArray();

        // 获取 Faker 实例
        $faker = app(Faker\Generator::class);

        $topics = factory(Topic::class)
                        ->times(100)
                        ->make()
                        ->each(function ($topic, $index)
                            use ($user_ids, $category_ids, $faker)
        {
            // 从用户 ID 数组中随机取出一个并赋值
            $topic->user_id = $faker->randomElement($user_ids);

            // 话题分类，同上
            $topic->category_id = $faker->randomElement($category_ids);
        });

        // 将数据集合转换为数组，并插入到数据库中
        Topic::insert($topics->toArray());
    }
}
```

### 4. 注册数据填充

我们需要将 DatabaseSeeder.php 中 TopicsTableSeeder 调用的注释去掉：

_database/seeds/DatabaseSeeder.php_

```
<?php

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        $this->call(UsersTableSeeder::class);
        $this->call(TopicsTableSeeder::class);
    }
}
```

请注意`run()`方法里的顺序，我们先生成用户数据，再生出话题数据。

## 三、执行数据填充命令

一切准备就绪，接下来开始调用数据填充命令，将假数据写入数据库。我们只需要 10 个测试用户，而此时我们数据库里已经有 10 个用户，故使用：

```
$ php artisan migrate:refresh --seed
```

执行成功后查看数据库：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/49XRH7fLA7.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/49XRH7fLA7.png!large)

浏览器打开话题列表页面 ——[http://larabbs.test/topics](http://larabbs.test/topics)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/5hMb4gH8Wl.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/5hMb4gH8Wl.png!large)

可以看到有数据了，但是页面非常凌乱，不用担心，我们将在下一章节中对此页面进行定制。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "填充用户和话题数据"
```



