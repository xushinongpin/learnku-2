## 说明

在我们开发帖子功能之前，需要准备好帖子分类，因为每一篇帖子都必须发布在某一个分类下，帖子依赖于分类。

我们将会为 Larabbs 项目准备以下分类：

* 分享 —— 分享创造，分享发现；
* 教程 —— 教程相关文章；
* 问答 —— 用户问答相关的帖子；
* 公告 —— 站点公告类型的帖子。

## 数据模型

我们需要创建『分类模型（Model）』来跟数据库进行交互，创建的命令很简单。但请记住，所有我们新建的模型文件都要统一放置在`app/Models`文件夹下，为此我们在创建一个新的模型对象时，需要在模型名称前面加上`Models`目录。

```
$ php artisan make:model Models/Category -m
```

`-m`选项意为顺便创建数据库迁移文件（Migration）。使用`git status`查看文件创建状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Zb9VBUb5BM.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Zb9VBUb5BM.png!large)

> 注意：`2018_12_23_102155_`是为了防止文件名冲突的时间戳前缀，你的文件名跟我不一致为预料之中，为了防止混淆，下面我们将使用`{timestamp}`代替。

### 1. 数据库迁移

打开`{timestamp}_create_categories_table.php`文件，修改如下：

_{timestamp}\_create\_categories\_table.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateCategoriesTable extends Migration
{
    public function up()
    {
        Schema::create('categories', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name')->index()->comment('名称');
            $table->text('description')->nullable()->comment('描述');
            $table->integer('post_count')->default(0)->comment('帖子数');
        });
    }

    public function down()
    {
        Schema::dropIfExists('categories');
    }
}
```

字段讲解：

* `name`
  —— 分类的名称，为字符串类型，
  `index()`
  方法是加上索引，为 MySQL 下的搜索优化，
  `comment()`
  方法能为表结构注释；
* `description`
  —— 分类的描述，为文本类型，
  `nullable()`
  方法允许字段为空；
* `post_count`
  —— 分类下的帖子数量，计数器，
  `default(0)`
  方法设置默认值为 0；

保存文件后运行迁移生成数据表：

```
$ php artisan migrate
```

打开数据库工具，刷新一下，即可看见我们新添加的数据表：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cskPZJMOxJ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cskPZJMOxJ.png!large)

### 2. 数据模型

每当我们创建完数据模型后，都需要设置 Category 的`$fillable`白名单属性，告诉程序那些字段是支持修改的：

_app/Models/Category.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Category extends Model
{
    protected $fillable = [
        'name', 'description',
    ];
}
```

### 3. 初始化分类数据

我们想要 LaraBBS 论坛软件在安装的时，就初始化本文最前面提到的四个分类。

面对数据库内容填充的需求，一般情况下我们会使用 Laravel 的[『数据填充 Seed』](https://learnku.com/docs/laravel/5.7/seeding)。可是在当下场景中，我们无法使用此功能。此一般是用来生成假数据，而现在我们需要生成的是项目的**初始化数据**，这些数据是项目运行的一部分，在生产环境下也会使用到，而数据填充只能在开发时使用。

虽然 Laravel 没有自带此类解决方案，不过数据迁移功能倒是比较不错的替代方案。在功能定位上，数据迁移也是项目的一部分，执行的时机刚好是在项目安装时。并且区分执行先后顺序，这确保了初始化数据发生在数据表结构创建完成后。

接下来我们使用命令生成数据迁移文件，作为**初始化数据**的迁移文件，我们定义命名规范为`seed_(数据库表名称)_data`：

```
$ php artisan make:migration seed_categories_data
```

打开生成的迁移文件，并使用以下代码替换：

_database/migrations/{timestamp}\_seed\_categories\_data.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class SeedCategoriesData extends Migration
{
    public function up()
    {
        $categories = [
            [
                'name'        => '分享',
                'description' => '分享创造，分享发现',
            ],
            [
                'name'        => '教程',
                'description' => '开发技巧、推荐扩展包等',
            ],
            [
                'name'        => '问答',
                'description' => '请保持友善，互帮互助',
            ],
            [
                'name'        => '公告',
                'description' => '站点公告',
            ],
        ];

        DB::table('categories')->insert($categories);
    }

    public function down()
    {
        DB::table('categories')->truncate();
    }
}
```

代码解析：

1. `up()`
   方法中使用
   `DB`
   类的
   `insert()`
   批量往数据表
   `categories`
   里插入数据
   `$categories`
   ；
2. `down()`
   在回滚迁移时会被调用，是
   `up()`
   方法的逆反操作。
   `truncate()`
   方法为清空
   `categories`
   数据表里的所有数据。

执行迁移：

```
$ php artisan migrate
```

迁移成功后，打开数据库工具即可看到我们的数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/K2MAHcs7mF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/K2MAHcs7mF.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "分类模型和数据"
```



