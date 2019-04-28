## 通知列表

本节课我们将一起制作通知列表页面，完善我们的评论通知功能。

## 1. 新建路由器

首先我们需要新建路由入口：

_routes/web.php_

```
.
.
.

Route::resource('notifications', 'NotificationsController', ['only' => ['index']]);
```

## 2. 顶部导航栏入口

我们希望用户在访问网站时，能在很显眼的地方提醒他你有未读信息，接下来我们会利用上`notification_count`字段，新增下面的`消息通知标记`区块：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
        <!-- Authentication Links -->
        @guest
          <li class="nav-item"><a class="nav-link" href="{{ route('login') }}">登录</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ route('register') }}">注册</a></li>
        @else
          <li class="nav-item">
            <a class="nav-link mt-1 mr-3 font-weight-bold" href="{{ route('topics.create') }}">
              <i class="fa fa-plus"></i>
            </a>
          </li>
          <li class="nav-item notification-badge">
            <a class="nav-link mr-3 badge badge-pill badge-{{ Auth::user()->notification_count > 0 ? 'hint' : 'secondary' }} text-white" href="{{ route('notifications.index') }}">
              {{ Auth::user()->notification_count }}
            </a>
          </li>
          <li class="nav-item dropdown">
.
.
.
```

刷新页面即可看到消息提醒标示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/vLxWNCipRy.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/vLxWNCipRy.png!large)

样式有点乱，我们稍加调整：

_resources/sass/app.scss_

```
.
.
.

/* 消息通知 */
.notification-badge {
  .badge {
    font-size: 12px;
    margin-top: 14px;
  }

  .badge-secondary {
    background-color: #EBE8E8;
  }

  .badge-hint {
    background-color: #d15b47 !important;
  }
}
```

默认情况下样式很低调：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/FaJwpDxBXP.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/FaJwpDxBXP.png!large)

我们重新使用 Summer 用户登录，可看到显眼的红色标示，并且带有未读消息数量：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/vJVbkCTDfE.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/vJVbkCTDfE.png!large)

## 3. 控制器

如果你点击红色标示，会报错 —— 控制器文件并不存在：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/ZjQcTDKz9b.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/ZjQcTDKz9b.png!large)

接下来我们使用命令行生成控制器：

```
$ php artisan make:controller NotificationsController
```

修改控制器的代码如下：

_app/Http/Controllers/NotificationsController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Auth;

class NotificationsController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
    }

    public function index()
    {
        // 获取登录用户的所有通知
        $notifications = Auth::user()->notifications()->paginate(20);
        return view('notifications.index', compact('notifications'));
    }
}
```

控制器的构造方法`__construct()`里调用 Auth 中间件，要求必须登录以后才能访问控制器里的所有方法。

## 4. 通知列表视图

再次刷新页面，你会看到视图文件未找到的异常：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/3nE0bKdPab.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/3nE0bKdPab.png!large)

接下来新建此模板：

_resources/views/notifications/index.blade.php_

```
@extends('layouts.app')

@section('title', '我的通知')

@section('content')
  <div class="container">
    <div class="col-md-10 offset-md-1">
      <div class="card ">

        <div class="card-body">

          <h3 class="text-xs-center">
            <i class="far fa-bell" aria-hidden="true"></i> 我的通知
          </h3>
          <hr>

          @if ($notifications->count())

            <div class="list-unstyled notification-list">
              @foreach ($notifications as $notification)
                @include('notifications.types._' . snake_case(class_basename($notification->type)))
              @endforeach

              {!! $notifications->render() !!}
            </div>

          @else
            <div class="empty-block">没有消息通知！</div>
          @endif

        </div>
      </div>
    </div>
  </div>
@stop
```

通知数据库表的 Type 字段保存的是通知类全称，如 ：App\Notifications\TopicReplied 。`snake_case(class_basename($notification->type))`渲染以后会是 ——`topic_replied`。`class_basename()`方法会取到`TopicReplied`，Laravel 的辅助方法`snake_case()`会字符串格式化为下划线命名。

刷新页面，会提示我们对应类型的模板文件不存在：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/WIH7FLtDWA.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/WIH7FLtDWA.png!large)

创建此文件：

_resources/views/notifications/types/\_topic\_replied.blade.php_

```
<li class="media @if ( ! $loop->last) border-bottom @endif">
  <div class="media-left">
    <a href="{{ route('users.show', $notification->data['user_id']) }}">
      <img class="media-object img-thumbnail mr-3" alt="{{ $notification->data['user_name'] }}" src="{{ $notification->data['user_avatar'] }}" style="width:48px;height:48px;" />
    </a>
  </div>

  <div class="media-body">
    <div class="media-heading mt-0 mb-1 text-secondary">
      <a href="{{ route('users.show', $notification->data['user_id']) }}">{{ $notification->data['user_name'] }}</a>
      评论了
      <a href="{{ $notification->data['topic_link'] }}">{{ $notification->data['topic_title'] }}</a>

      {{-- 回复删除按钮 --}}
      <span class="meta float-right" title="{{ $notification->created_at }}">
        <i class="far fa-clock"></i>
        {{ $notification->created_at->diffForHumans() }}
      </span>
    </div>
    <div class="reply-content">
      {!! $notification->data['reply_content'] !!}
    </div>
  </div>
</li>
```

我们可以通过`$notification->data`拿到在通知类`toDatabase()`里构建的数组。

刷新页面即可看到我们的消息通知列表：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/WtsA8GOZYR.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/WtsA8GOZYR.png!large)

## 5. 清除未读消息标示

下面我们来开发去除顶部未读消息标示的功能 —— 当用户访问通知列表时，将所有通知状态设定为已读，并清空未读消息数。

接下来在 User 模型中新增`markAsRead()`方法：

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

    public function markAsRead()
    {
        $this->notification_count = 0;
        $this->save();
        $this->unreadNotifications->markAsRead();
    }
}
```

修改控制器的`index()`方法，新增清空未读提醒的状态：

_app/Http/Controllers/NotificationsController.php_

```
<?php
.
.
.
class NotificationsController extends Controller
{
    .
    .
    .
    public function index()
    {
        // 获取登录用户的所有通知
        $notifications = Auth::user()->notifications()->paginate(20);
        // 标记为已读，未读数量清零
        Auth::user()->markAsRead();
        return view('notifications.index', compact('notifications'));
    }
}
```

现在进入消息通知页面，未读消息标示将被清除：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/kngxqfBZcw.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/kngxqfBZcw.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "消息通知列表"
```



