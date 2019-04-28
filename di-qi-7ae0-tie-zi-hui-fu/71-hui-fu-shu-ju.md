## 回复列表

目前我们已经有话题发布功能，接下来我们一起开发话题回复功能，允许用户参与讨论某个话题。回复功能包括：

* 回复列表
* 发表回复
* 删除回复
* 消息通知

本章节，我们先开发『回复列表』功能。

## 代码生成

使用代码生成器快速构建骨架代码：

```
$ php artisan make:scaffold Reply --schema="topic_id:integer:unsigned:default(0):index,user_id:integer:unsigned:default(0):index,content:text"
```

## 数据模型

修改下 Reply 模型的`$fillable`属性，我们只允许用户更改`content`字段。同时做下数据模型的关联，一条回复属于一个话题，一个条回复属于一个作者所有：

_app/Models/Reply.php_

```
<?php

namespace App\Models;

class Reply extends Model
{
    protected $fillable = ['content'];

    public function topic()
    {
        return $this->belongsTo(Topic::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

接下来修改对应关联 Topic 和 User 模型，新增对 Reply 的所属关系。

一篇帖子下有多条回复，新增`replies()`方法：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
    protected $fillable = [
        'title', 'body', 'category_id', 'excerpt', 'slug'
    ];

    public function replies()
    {
        return $this->hasMany(Reply::class);
    }

    .
    .
    .
}
```

一个用户可以拥有多条评论，新增`replies()`方法：

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

    public function replies()
    {
        return $this->hasMany(Reply::class);
    }
}
```

## 假数据生成

接下来我们将要填充回复假数据，以便回复列表的开发。

### 1. 定制数据工厂

_database/factories/ReplyFactory.php_

```
<?php

use Faker\Generator as Faker;

$factory->define(App\Models\Reply::class, function (Faker $faker) {

    // 随机取一个月以内的时间
    $time = $faker->dateTimeThisMonth();

    return [
        'content' => $faker->sentence(),
        'created_at' => $time,
        'updated_at' => $time,
    ];
});
```

### 2. 数据填充逻辑

_database/seeds/ReplysTableSeeder.php_

```
<?php

use Illuminate\Database\Seeder;
use App\Models\Reply;
use App\Models\User;
use App\Models\Topic;

class ReplysTableSeeder extends Seeder
{
    public function run()
    {
        // 所有用户 ID 数组，如：[1,2,3,4]
        $user_ids = User::all()->pluck('id')->toArray();

        // 所有话题 ID 数组，如：[1,2,3,4]
        $topic_ids = Topic::all()->pluck('id')->toArray();

        // 获取 Faker 实例
        $faker = app(Faker\Generator::class);

        $replys = factory(Reply::class)
                        ->times(1000)
                        ->make()
                        ->each(function ($reply, $index)
                            use ($user_ids, $topic_ids, $faker)
        {
            // 从用户 ID 数组中随机取出一个并赋值
            $reply->user_id = $faker->randomElement($user_ids);

            // 话题 ID，同上
            $reply->topic_id = $faker->randomElement($topic_ids);
        });

        // 将数据集合转换为数组，并插入到数据库中
        Reply::insert($replys->toArray());
    }
}
```

### 3. 注册数据填充类

代码生成器已经为我们注册了数据填充类，不过顺序有错，我们需要修改下顺序，TopicsTableSeeder 应在 ReplysTableSeeder 之前执行：

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
        $this->call(ReplysTableSeeder::class);
    }
}
```

### 4. 开始填充数据

使用以下命令重新填充数据：

```
$ php artisan migrate:refresh --seed
```

打开数据库图形工具即可看到生成内容，每个回复都有对应的话题 ID 和用户 ID：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/csPFYo8MUY.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/csPFYo8MUY.png!large)

下一章节我们要将这些数据显示出来。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "回复数据"
```



