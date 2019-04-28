## 删除回复

接下来我们将允许回复的作者和话题的作者删除回复。删除的动作发生在话题详情页下方的回复列表里。

## 1. 删除按钮

首先修改下模板，将之前的`<a></a>`标签修改为删除表单按钮：

_resources/views/topics/\_reply\_list.blade.php_

```
.
.
.
          {{-- 回复删除按钮 --}}
          <span class="meta float-right">
            <form action="{{ route('replies.destroy', $reply->id) }}"
                onsubmit="return confirm('确定要删除此评论？');"
                method="post">
              {{ csrf_field() }}
              {{ method_field('DELETE') }}
              <button type="submit" class="btn btn-default btn-xs pull-left text-secondary">
                <i class="far fa-trash-alt"></i>
              </button>
            </form>
          </span>
.
.
.
```

## 2. 控制器处理

随便进入一个拥有评论的话题里，点击评论删除按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/2LCe268uL5.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/2LCe268uL5.png!large)

确定提交后会报错：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/lMQT382pYA.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/lMQT382pYA.png!large)

查看控制器，原来是下面这一行有问题，成功删除后跳转到`index`，而此页面不存在，所以报错：

```
return redirect()->route('replies.index')->with('success', '删除成功！');
```

重新修改`destroy()`方法：

_app/Http/Controllers/RepliesController.php_

```
<?php
.
.
.
class RepliesController extends Controller
{
    .
    .
    .

    public function destroy(Reply $reply)
    {
        $this->authorize('destroy', $reply);
        $reply->delete();

        return redirect()->to($reply->topic->link())->with('success', '评论删除成功！');
    }
}
```

再次尝试一遍：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/hjfPv7xID0.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/hjfPv7xID0.gif!large)

## 3. 权限控制

目前我们的删除功能并没有做限制，只要是登录用户即可使用，当然这不是我们想要的。拥有删除回复权限的用户，应当是『回复的作者』或者『回复话题的作者』：

_app/Policies/ReplyPolicy.php_

```
<?php

namespace App\Policies;

use App\Models\User;
use App\Models\Reply;

class ReplyPolicy extends Policy
{
    public function destroy(User $user, Reply $reply)
    {
        return $user->isAuthorOf($reply) || $user->isAuthorOf($reply->topic);
    }
}
```

我们自定义的辅助方法`isAuthorOf()`极大提高了代码的可读性。

控制器`destroy()`方法中已经做了如下调用，此处我们无需修改：

```
$this->authorize('destroy', $reply);
```

### 删除按钮显示逻辑

我们还需要对视图文件进行处理，只有当用户拥有删除回复权限时，才显示按钮。在 Blade 模板里我们可以使用`@can`语句来实现：

_resources/views/topics/\_reply\_list.blade.php_

```
.
.
.
          {{-- 回复删除按钮 --}}
          @can('destroy', $reply)
            <span class="meta float-right">
              <form action="{{ route('replies.destroy', $reply->id) }}"
                  onsubmit="return confirm('确定要删除此评论？');"
                  method="post">
                {{ csrf_field() }}
                {{ method_field('DELETE') }}
                <button type="submit" class="btn btn-default btn-xs pull-left text-secondary">
                  <i class="far fa-trash-alt"></i>
                </button>
              </form>
            </span>
          @endcan
.
.
.
```

测试一下，目前是两种逻辑。

第一种，寻找当前登录用户 Summer 发布的帖子，可以看到所有回复都有删除按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/JogOOPTMYJ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/JogOOPTMYJ.png!large)

第二种，寻找其他用户发布的帖子，发布一个评论，可以看到只能删除自己发表的评论：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/mIZinLj5Wd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/mIZinLj5Wd.png!large)

## 4. 减去话题回复数

我们还需要对话题的回复数进行跟踪，当新增话题回复时，我们已经对话题的`reply_count`做了更新操作，同样的，当回复被删除后，评论数已变更，话题的`reply_count`也需要更新。我们需要在模型监听器中添加`deleted()`方法来监控『回复删除后』事件：

_app/Observers/ReplyObserver.php_

```
<?php
.
.
.
class ReplyObserver
{
    .
    .
    .
    public function deleted(Reply $reply)
    {
        $reply->topic->reply_count = $reply->topic->replies->count();
        $reply->topic->save();
    }
}
```

发现了一段重复的代码，为了可维护性更佳，我们必须将其抽象化出来：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
.
.
.

    public function updateReplyCount()
    {
        $this->reply_count = $this->replies->count();
        $this->save();
    }

}
```

重构之后的代码：

_app/Observers/ReplyObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Reply;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

use App\Notifications\TopicReplied;

class ReplyObserver
{
    public function created(Reply $reply)
    {
        $reply->topic->updateReplyCount();
        // 通知话题作者有新的评论
        $reply->topic->user->notify(new TopicReplied($reply));
    }

    public function creating(Reply $reply)
    {
        $reply->content = clean($reply->content, 'user_topic_body');
    }

    public function deleted(Reply $reply)
    {
        $reply->topic->updateReplyCount();
    }
}
```

## 5. 话题连带删除

在我们的业务逻辑中，回复是针对话题而存在的。当话题被删除的时候，数据库里的回复信息没有存在的价值，只会占用空间。所以接下来我们将监听话题删除成功的事件，在此事件发生时，我们会删除此话题下所有的回复：

_app/Observers/TopicObserver.php_

```
<?php
.
.
.
class TopicObserver
{
    .
    .
    .

    public function deleted(Topic $topic)
    {
        \DB::table('replies')->where('topic_id', $topic->id)->delete();
    }
}
```

新增了`deleted()`方法来监控话题成功删除的事件。需注意，在模型监听器中，数据库操作需避免再次触发 Eloquent 事件，以免造成联动逻辑冲突。所以这里我们使用了 DB 类进行操作。

测试一下删除话题后，是否回复同时被删除：

1. 找到一篇当前登录用户发布的文章，从浏览器中获取 ID ：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/8S2PsgaZ1F.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/8S2PsgaZ1F.png!large)

1. 数据库里定位到此篇文章的评论：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/qg2Y5pBeBX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/qg2Y5pBeBX.png!large)

1. 删除文章后刷新数据库：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/spv9da8AsQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/spv9da8AsQ.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "删除评论"
```



