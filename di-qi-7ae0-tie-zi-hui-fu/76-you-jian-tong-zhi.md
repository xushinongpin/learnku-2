## 邮件通知

本章节我们将为『话题有新回复』通知新增『邮件通知频道』，当话题被回复时，作者可以收到一份 Email 邮件通知。Laravel 的通知系统默认支持邮件频道的通知方式，我们只需要稍作配置即可。

Laravel 支持多种邮件驱动，包括 smtp、Mailgun、Maildrill、Amazon SES、mail 和 sendmail。Mailgun 、 Amazon SES 、Maildrill 都是第三方邮件服务。mail 驱动使用 PHP 提供的 mail 函数。sendmail 驱动通过 Sendmail/Postfix（Linux）提供的命令发送邮件，smtp 驱动使用支持 ESMTP 的 SMTP 服务器发送邮件。mail 不安全，sendmail 需要安装配置 Sendmail/Postfix，并且信用不高，很容易被当成垃圾邮件，第三方服务的信用是最高的，商业软件推荐使用。

本章节以 QQ 邮箱为例，我们将开启 QQ 的 SMTP 功能，并配置项目的 SMTP 邮件发送功能。其他邮箱的配置基本大致相同。

## 1. 开启 QQ 邮箱的 SMTP 支持

首先我们需要在 QQ 邮箱的账号设置里开启 POP3 和 SMTP 服务。具体请查看[如何打开 POP3/SMTP/IMAP 功能？](http://service.mail.qq.com/cgi-bin/help?subtype=1&id=28&no=166)。

只需要开启以下：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/6vbb8bto8O.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/6vbb8bto8O.png)

复制方框里的『授权码』，授权码将作为我们的密码使用：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/POcuxq8WDd.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/POcuxq8WDd.png)

## 2. 邮箱发送配置

Laravel 中邮箱发送的配置存放于`config/mail.php`中。不过`mail.php`中我们所需的配置，都可以通过`.env`来配置。作为最佳实践，我们优先选择通过环境变量来配置：

_.env_

```
.
.
.
MAIL_DRIVER=smtp
MAIL_HOST=smtp.qq.com
MAIL_PORT=25
MAIL_USERNAME=xxxxxxxxxxxxxx@qq.com
MAIL_PASSWORD=xxxxxxxxx
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=xxxxxxxxxxxxxx@qq.com
MAIL_FROM_NAME=LaraBBS
.
.
.
```

选项讲解：

1. MAIL\_DRIVER=smtp —— 使用支持 ESMTP 的 SMTP 服务器发送邮件；
2. MAIL\_HOST=smtp.qq.com —— QQ 邮箱的 SMTP 服务器地址，必须为此值；
3. MAIL\_PORT=25 —— QQ 邮箱的 SMTP 服务器端口，必须为此值；
4. MAIL\_USERNAME=xxxxxxxxxxxxxx@qq.com —— 请将此值换为你的 QQ +
   `@qq.com`
   ；
5. MAIL\_PASSWORD=xxxxxxxxx —— 密码是我们第一步拿到的授权码；
6. MAIL\_ENCRYPTION=tls —— 加密类型，选项 null 表示不使用任何加密，其他选项还有 ssl，这里我们使用 tls 即可。
7. MAIL\_FROM\_ADDRESS=xxxxxxxxxxxxxx@qq.com —— 此值必须同 MAIL\_USERNAME 一致；
8. MAIL\_FROM\_NAME=LaraBBS —— 用来作为邮件的发送者名称。

## 3. 添加邮件通知频道

首先我们需要修改`via()`方法，并新增`mail`通知频道：

_app/Notifications/TopicReplied.php_

```
<?php
.
.
.
class TopicReplied extends Notification
{
    .
    .
    .
    public function via($notifiable)
    {
        // 开启通知的频道
        return ['database', 'mail'];
    }
    .
    .
    .
}
```

因为开启了`mail`频道，我们还需要新增`toMail`方法：

_app/Notifications/TopicReplied.php_

```
<?php
.
.
.
class TopicReplied extends Notification
{
    .
    .
    .

    public function toMail($notifiable)
    {
        $url = $this->reply->topic->link(['#reply' . $this->reply->id]);

        return (new MailMessage)
                    ->line('你的话题有新回复！')
                    ->action('查看回复', $url);
    }
}
```

## 4. 测试邮件通知

首先我们利用数据库工具修改 ID 为 1 的用户邮箱为你自己的 QQ 邮箱（或者你自己的其他邮箱账号也可），这样我们才能更加直观地检测邮件是否发送成功：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/28CzwMac8F.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/28CzwMac8F.png)

登录除了 ID 为 1 的用户，并寻找 Summer 发布的话题进行回复提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/9hjxH1QlKn.png)](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/9hjxH1QlKn.png)

提交成功后，刷新邮箱，一般几分钟内就能收到话题回复的邮件：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/9rwfTZnlR3.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/9rwfTZnlR3.png)

## 5. 使用队列发送邮件

大家应该会发现我们提交回复时，服务器响应会变得非常缓慢，这是『邮件通知』功能请求了 QQ SMTP 服务器进行邮件发送所产生的延迟。我们已经学过了，对于处理此类延迟，最好的方式是使用队列系统。

我们可以通过对通知类添加 ShouldQueue 接口和 Queueable trait 把通知加入队列。它们两个在使用`make:notification`命令来生成通知文件时就已经被导入，我们只需添加到通知类接口即可。

修改`TopicReplied.php`文件，将以下这一行：

```
class TopicReplied extends Notification
```

改为：

```
class TopicReplied extends Notification implements ShouldQueue
```

Laravel 会检测`ShouldQueue`接口并自动将通知的发送放入队列中，所以我们不需要做其他修改。

### 测试下队列

将 QUEUE\_CONNECTION 的值改为 redis：

_.env_

```
QUEUE_CONNECTION=redis
```

命令行运行队列监控：

```
$ php artisan horizon
```

发送邮件测试，可以发现速度大大提高，队列监控也接收到队列任务，并成功处理：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/3URvP6JMBe.gif "file")](https://iocaffcdn.phphub.org/uploads/images/201710/23/1/3URvP6JMBe.gif)

测试成功后，为了简化后续的开发，我们将配置改为原来的 sync 实时模式：

_.env_

```
QUEUE_CONNECTION=sync
```

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "支持队列的邮件通知"
```



