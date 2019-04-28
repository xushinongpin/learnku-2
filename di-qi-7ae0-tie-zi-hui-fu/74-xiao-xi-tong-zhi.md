## 消息通知

接下来我们开发消息通知功能，当话题有新回复时，我们将通知作者『你的话题有新回复，请查看』类似的信息。

## Laravel 的消息通知系统

Laravel 自带了一套极具扩展性的消息通知系统，尤其还支持多种通知频道，我们将利用此套系统来向用户发送消息提醒。

### 什么是通知频道？

通知频道是通知传播的途径，Laravel 自带的有数据库、邮件、短信（通过 Nexmo）以及 Slack。本章节中我们将使用数据库通知频道，后面也会使用到邮件通知频道。

## 1. 准备数据库

数据通知频道会在一张数据表里存储所有通知信息。包含了比如通知类型、JSON 格式数据等描述通知的信息。我们后面会通过查询这张表的内容在应用界面上展示通知。但是在这之前，我们需要先创建这张数据表，Laravel 自带了生成迁移表的命令，执行以下命令即可：

```
$ php artisan notifications:table
```

会生成`database/migrations/{$timestamp}_create_notifications_table.php`迁移文件，执行`migrate`命令将表结构写入数据库中：

```
$ php artisan migrate
```

我们还需要在`users`表里新增`notification_count`字段，用来跟踪用户有多少未读通知，如果未读通知大于零的话，就在站点的全局顶部导航栏显示红色的提醒。

```
$ php artisan make:migration add_notification_count_to_users_table --table=users
```

打开生成的文件，修改为以下：

_database/migrations/{$timestamp}\_add\_notification\_count\_to\_users\_table.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddNotificationCountToUsersTable extends Migration
{
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->integer('notification_count')->unsigned()->default(0);
        });
    }

    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('notification_count');
        });
    }
}
```

再次应用数据库修改：

```
$ php artisan migrate
```

## 2. 生成通知类

Laravel 中一条通知就是一个类（通常存在 app/Notifications 文件夹里）。看不到的话不要担心，运行一下以下命令即可创建：

```
$ php artisan make:notification TopicReplied
```

修改文件为以下：

_app/Notifications/TopicReplied.php_

```
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use App\Models\Reply;

class TopicReplied extends Notification
{
    use Queueable;

    public $reply;

    public function __construct(Reply $reply)
    {
        // 注入回复实体，方便 toDatabase 方法中的使用
        $this->reply = $reply;
    }

    public function via($notifiable)
    {
        // 开启通知的频道
        return ['database'];
    }

    public function toDatabase($notifiable)
    {
        $topic = $this->reply->topic;
        $link =  $topic->link(['#reply' . $this->reply->id]);

        // 存入数据库里的数据
        return [
            'reply_id' => $this->reply->id,
            'reply_content' => $this->reply->content,
            'user_id' => $this->reply->user->id,
            'user_name' => $this->reply->user->name,
            'user_avatar' => $this->reply->user->avatar,
            'topic_link' => $link,
            'topic_id' => $topic->id,
            'topic_title' => $topic->title,
        ];
    }
}
```

每个通知类都有个`via()`方法，它决定了通知在哪个频道上发送。我们写上`database`数据库来作为通知频道。

因为使用数据库通知频道，我们需要定义`toDatabase()`。这个方法接收`$notifiable`实例参数并返回一个普通的 PHP 数组。这个返回的数组将被转成 JSON 格式并存储到通知数据表的`data`字段中。

## 3. 触发通知

我们希望当用户回复主题后，通知到主题作者。故触发通知的时机是：『回复发布成功后』，在模型监控器里，我们可以在`created`方法里实现此部分代码，修改`created()`方法为以下：

_app/Observers/ReplyObserver.php_

```
<?php
.
.
.

use App\Notifications\TopicReplied;

class ReplyObserver
{
    public function created(Reply $reply)
    {
        $reply->topic->reply_count = $reply->topic->replies->count();
        $reply->topic->save();

        // 通知话题作者有新的评论
        $reply->topic->user->notify(new TopicReplied($reply));
    }

    .
    .
    .
}
```

请注意顶部引入`TopicReplied`。默认的`User`模型中使用了 trait —— Notifiable，它包含着一个可以用来发通知的方法`notify()`，此方法接收一个通知实例做参数。虽然`notify()`已经很方便，但是我们还需要对其进行定制，我们希望每一次在调用`$user->notify()`时，自动将`users`表里的`notification_count`+1 ，这样我们就能跟踪用户未读通知了。

打开`User.php`文件，将`use Notifiable, MustVerifyEmailTrait;`修改为以下：

_app/Models/User.php_

```
<?php
.
.
.

use Auth;

class User extends Authenticatable implements MustVerifyEmailContract
{
    use MustVerifyEmailTrait;

    use Notifiable {
        notify as protected laravelNotify;
    }
    public function notify($instance)
    {
        // 如果要通知的人是当前用户，就不必通知了！
        if ($this->id == Auth::id()) {
            return;
        }

        // 只有数据库类型通知才需提醒，直接发送 Email 或者其他的都 Pass
        if (method_exists($instance, 'toDatabase')) {
            $this->increment('notification_count');
        }

        $this->laravelNotify($instance);
    }

    .
    .
    .
}
```

> 请注意顶部 Auth 的引入。

我们对`notify()`方法做了一个巧妙的重写，现在每当你调用`$user->notify()`时，`users`表里的`notification_count`将自动 +1。

### 测试一下

我们先退出 Summer 用户的登录，然后从数据库里随便拿另一个用户的邮箱：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/dPnjRBCUDz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/dPnjRBCUDz.png!large)

访问登录页面，黏贴进去邮箱，密码是通用的`secret`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/mCGTr0S4aa.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/mCGTr0S4aa.png!large)

登录成功后，访问 Summer 的个人空间[http://larabbs.test/users/1](http://larabbs.test/users/1)，找一条 Summer 发布过的话题进行回复：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/nciNbJGnU8.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/nciNbJGnU8.png!large)

刷新数据库即可看到`notifications`表里多了一条数据，并且`notifiable_id`为 1 ，也就是 1 号用户 Summer 有一条新通知：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/syoLj8oV0H.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/syoLj8oV0H.png!large)

下一节我们将制作通知列表页面，将数据库里的消息通知展示出来。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "消息通知有新回复"
```



