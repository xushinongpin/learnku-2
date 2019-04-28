## 使用队列

上一章节中我们开发了自动生成 Slug 功能，但是因为我们的需要实时请求百度翻译接口，这将会是一个系统性能隐患。

一般情况下，网络请求会存在各种不确定性，如果请求 API 出现超时情况，或者发生不可预知的错误，我们的用户将无法发帖。

生成 Slug 只是一个**优化**功能，并非是发帖的**必要**功能，我们希望无论生成 Slug 的结果如何，用户都能顺利的发帖，并且完全察觉不到延迟。

利用队列系统可以做到这点。队列允许你异步执行消耗时间的任务，比如请求一个 API 并等待返回的结果。这样可以有效的降低请求响应的时间。

## 1. 配置队列

队列的配置信息储存于`config/queue.php`文件中，在这个文件中你会发现框架所支持的队列驱动的配置连接示例。这些驱动包括：数据库，Beanstalkd，Amazon SQS，Redis，和一个同步（本地使用）的驱动。还有一个名为`null`的驱动表明不使用队列任务。

本项目中，我们将使用 Redis 来作为我们的队列驱动器，先使用 Composer 安装依赖：

```
$ composer require "predis/predis:~1.1"
```

修改环境变量`QUEUE_DRIVER`的值为`redis`：

_.env_

```
.
.
.
QUEUE_CONNECTION=redis
.
.
.
```

### 失败任务

有时候队列中的任务会失败。Laravel 内置了一个方便的方式来指定任务重试的最大次数。当任务超出这个重试次数后，它就会被插入到`failed_jobs`数据表里面。我们可以使用`queue:failed-table`命令来创建`failed_jobs`表的迁移文件：

```
$ php artisan queue:failed-table
```

会新建`database/migrations/{timestamp}_create_failed_jobs_table.php`文件：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/uSJBz9v17T.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/uSJBz9v17T.png)

接着使用`migrate`Artisan 命令生成`failed_jobs`表：

```
$ php artisan migrate
```

## 2. 生成任务类

使用以下 Artisan 命令来生成一个新的队列任务：

```
$ php artisan make:job TranslateSlug
```

该命令会在`app/Jobs`目录下生成一个新的类：

_app/Jobs/TranslateSlug.php_

```
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

use App\Models\Topic;
use App\Handlers\SlugTranslateHandler;

class TranslateSlug implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $topic;

    public function __construct(Topic $topic)
    {
        // 队列任务构造器中接收了 Eloquent 模型，将会只序列化模型的 ID
        $this->topic = $topic;
    }

    public function handle()
    {
        // 请求百度 API 接口进行翻译
        $slug = app(SlugTranslateHandler::class)->translate($this->topic->title);

        // 为了避免模型监控器死循环调用，我们使用 DB 类直接对数据库进行操作
        \DB::table('topics')->where('id', $this->topic->id)->update(['slug' => $slug]);
    }
}
```

该类实现了`Illuminate\Contracts\Queue\ShouldQueue`接口，该接口表明 Laravel 应该将该任务添加到后台的任务队列中，而不是同步执行。

引入了`SerializesModels`trait，Eloquent 模型会被优雅的序列化和反序列化。队列任务构造器中接收了 Eloquent 模型，将会只序列化模型的 ID。这样子在任务执行时，队列系统会从数据库中自动的根据 ID 检索出模型实例。这样可以避免序列化完整的模型可能在队列中出现的问题。

`handle`方法会在队列任务执行时被调用。值得注意的是，我们可以在任务的`handle`方法中可以使用类型提示来进行依赖的注入。Laravel 的服务容器会自动的将这些依赖注入进去，与控制器方法类似。

还有一点需要注意，我们将会在模型监控器中分发任务，任务中要避免使用 Eloquent 模型接口调用，如：`create()`,`update()`,`save()`等操作。否则会陷入调用死循环 —— 模型监控器分发任务，任务触发模型监控器，模型监控器再次分发任务，任务再次触发模型监控器.... 死循环。在这种情况下，使用`DB`类直接对数据库进行操作即可。

## 3. 任务分发

接下来我们要修改 Topic 模型监控器，将 Slug 翻译的调用修改为队列执行的方式：

_app/Observers/TopicObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Topic;
use App\Jobs\TranslateSlug;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class TopicObserver
{
    public function saving(Topic $topic)
    {
        // XSS 过滤
        $topic->body = clean($topic->body, 'user_topic_body');

        // 生成话题摘录
        $topic->excerpt = make_excerpt($topic->body);

        // 如 slug 字段无内容，即使用翻译器对 title 进行翻译
        if ( ! $topic->slug) {

            // 推送任务到队列
            dispatch(new TranslateSlug($topic));
        }
    }
}
```

## 4. 开始测试

开始之前，我们需要在命令行启动队列系统，队列在启动完成后会进入监听状态：

```
$ php artisan queue:listen
```

浏览器打开话题发布页面，填写测试内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/hcGD1rZruq.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/hcGD1rZruq.png!large)

点击『保存』按钮提交表单后，可以在命令行中看到监听的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/zUotHMe3yc.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/zUotHMe3yc.png!large)

可以看到我们的任务`Failed`执行失败了。打开数据库查看`failed_jobs`里的数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/uWTrQceH1J.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/uWTrQceH1J.png!large)

虽然我们能够从`payload`和`exception`字段中看到报错的信息，但因为是序列化以后的信息，所以并不直观：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/FBxb9qJETR.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/FBxb9qJETR.png!large)

接下来我们将寻找更好的队列监控方案。

## 5. 队列监控 Horizon

[![](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/0jZWSe5fzC.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/0jZWSe5fzC.png)

[Horizon](https://learnku.com/docs/laravel/5.7/horizon)是 Laravel 生态圈里的一员，为 Laravel Redis 队列提供了一个漂亮的仪表板，允许我们很方便地查看和管理 Redis 队列任务执行的情况。

使用 Composer 安装：

```
$ composer require "laravel/horizon:~1.3"
```

安装完成后，使用`vendor:publish`Artisan 命令发布相关文件：

```
$ php artisan vendor:publish --provider="Laravel\Horizon\HorizonServiceProvider"
```

分别是配置文件`config/horizon.php`和存放在`public/vendor/horizon`文件夹中的 CSS 、JS 等页面资源文件。

至此安装完毕，浏览器打开[http://larabbs.test/horizon](http://larabbs.test/horizon)访问控制台：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/5kB4y4CujR.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/5kB4y4CujR.png!large)

Horizon 是一个监控程序，需要常驻运行，我们可以通过以下命令启动：

```
$ php artisan horizon
```

安装了 Horizon 以后，我们将使用`horizon`命令来启动队列系统和任务监控，无需使用`queue:listen`。

接下来我们再次尝试下发帖，发帖之前，请确保`horizon`命令处于监控状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/Zbchnd8qIU.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/Zbchnd8qIU.gif!large)

这一次多亏了 Horizon，我们可以清晰的看到更加详尽的错误信息，错误异常是`ModelNotFoundException`，最重要的：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/vtdU4joDwz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/vtdU4joDwz.png!large)

我们发现 Data 区块里，`id`的值居然为`null`。我们知道的，队列系统对于构造器里传入的 Eloquent 模型，将会只序列化 ID 字段，因为我们是在 Topic 模型监控器的`saving()`方法中分发队列任务的，此时传参的`$topic`变量还未在数据库里创建，所以`$topic->id`为 null。

## 6. 代码调整

既然我们已经定位到了问题，解决的方法也很简单，只需要确保分发任务时`$topic->id`有值即可。我们需要修改任务分发的时机：

_app/Observers/TopicObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Topic;
use App\Jobs\TranslateSlug;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class TopicObserver
{
    public function saving(Topic $topic)
    {
        // XSS 过滤
        $topic->body = clean($topic->body, 'user_topic_body');

        // 生成话题摘录
        $topic->excerpt = make_excerpt($topic->body);
    }

    public function saved(Topic $topic)
    {
        // 如 slug 字段无内容，即使用翻译器对 title 进行翻译
        if ( ! $topic->slug) {

            // 推送任务到队列
            dispatch(new TranslateSlug($topic));
        }
    }
}
```

模型监控器的`saved()`方法对应 Eloquent 的`saved`事件，此事件发生在创建和编辑时、数据入库以后。在`saved()`方法中调用，确保了我们在分发任务时，`$topic->id`永远有值。

> 需要注意的是，`artisan horizon`队列工作的守护进程是一个常驻进程，它不会在你的代码改变时进行重启，当我们修改代码以后，需要在命令行中对其进行重启操作。

重启`horizon`命令后再次尝试，即可看到成功运行的队列：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/LffTvt2rDy.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/LffTvt2rDy.png!large)

## 7. 线上部署须知

在开发环境中，我们为了测试方便，直接在命令行里调用`artisan horizon`进行队列监控。然而在生产环境中，我们需要配置一个进程管理工具来监控`artisan horizon`命令的执行，以便在其意外退出时自动重启。当服务器部署新代码时，需要终止当前 Horizon 主进程，然后通过进程管理工具来重启，从而使用最新的代码。

简而言之，生产环境下使用队列需要注意以下两个问题：

1. 使用 Supervisor 进程工具进行管理，配置和使用请参照
   [文档](https://learnku.com/docs/laravel/5.7/horizon#Supervisor-%E9%85%8D%E7%BD%AE)
   进行配置；
2. 每一次部署代码时，需
   `artisan horizon:terminate`
   然后再
   `artisan horizon`
   重新加载代码。

线上部署的话，还要注意一个权限控制的问题，我们将在后面章节中讲到。

## 8. 使用 Sync 队列驱动

既然功能已经开发测试完毕，为了后续开发的方便，我们将开发环境的队列驱动改回`sync`同步模式，也就是说不使用任何队列，实时执行：

_.env_

```
.
.
.
QUEUE_CONNECTION=sync
.
.
.
```

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "翻译队列化"
```



