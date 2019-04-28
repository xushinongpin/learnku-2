## 创建话题回复

本章节我们将开发话题回复功能，允许用户对话题进行评论。

## 1. 构建回复表单

在开发话题列表时，我们创建了空文件`_reply_box.blade.php`，并在话题详情页中对其进行了加载：

```
@include('topics._reply_box', ['topic' => $topic])
```

话题回复功能我们只允许登录用户使用，未登录用户不显示即可。Laravel Blade 模板提供了一个『视条件加载子模板』的语法：

```
@includeWhen($boolean, 'view.name', ['some' => 'data'])
```

刚好适用我们的使用场景，请将`@include('topics._reply_box', ['topic' => $topic])`修改为以下：

_resources/views/topics/show.blade.php_

```
.
.
.
      {{-- 用户回复列表 --}}
      <div class="card topic-reply mt-4">
          <div class="card-body">
              @includeWhen(Auth::check(), 'topics._reply_box', ['topic' => $topic])
              @include('topics._reply_list', ['replies' => $topic->replies()->with('user')->get()])
          </div>
      </div>
.
.
.
```

现在我们可以往子模板填入内容：

_resources/views/topics/\_reply\_box.blade.php_

```
@include('shared._error')

<div class="reply-box">
  <form action="{{ route('replies.store') }}" method="POST" accept-charset="UTF-8">
    <input type="hidden" name="_token" value="{{ csrf_token() }}">
    <input type="hidden" name="topic_id" value="{{ $topic->id }}">
    <div class="form-group">
      <textarea class="form-control" rows="3" placeholder="分享你的见解~" name="content"></textarea>
    </div>
    <button type="submit" class="btn btn-primary btn-sm"><i class="fa fa-share mr-1"></i> 回复</button>
  </form>
</div>
<hr>
```

刷新页面即可看到我们的回复框：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/krwnbRyILT.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/krwnbRyILT.png!large)

退出登录后的情况：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/L3NAZALRiu.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/L3NAZALRiu.png!large)

## 2. 整理路由器

代码生成器为我们生成了完整的资源路由，不过我们只需要`store`和`destroy`的路由。因此我们需要修改 routes/web.php ，将以下：

```
Route::resource('replies', 'RepliesController', ['only' => ['index', 'show', 'create', 'store', 'update', 'edit', 'destroy']]);
```

修改为：

```
Route::resource('replies', 'RepliesController', ['only' => ['store', 'destroy']]);
```

## 3. 控制器处理请求

同样的，控制器中我们只需要留下`store`和`destroy`方法：

_app/Http/Controllers/RepliesController.php_

```
<?php

namespace App\Http\Controllers;

use App\Models\Reply;
use Illuminate\Http\Request;
use App\Http\Controllers\Controller;
use App\Http\Requests\ReplyRequest;
use Auth;

class RepliesController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
    }

    public function store(ReplyRequest $request, Reply $reply)
    {
        $reply->content = $request->content;
        $reply->user_id = Auth::id();
        $reply->topic_id = $request->topic_id;
        $reply->save();

        return redirect()->to($reply->topic->link())->with('success', '评论创建成功！');
    }

    public function destroy(Reply $reply)
    {
        $this->authorize('destroy', $reply);
        $reply->delete();

        return redirect()->route('replies.index')->with('success', '评论删除成功！');
    }
}
```

测试评论：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/PTIQDsQVLf.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/PTIQDsQVLf.gif!large)

## 4. 表单验证

一个开发的好习惯是：涉及开发表单提交时，都需要问下，我们必须对哪些数据进行验证，请记住 ——『绝不信任用户的输入』。目前的评论框存在一个问题，用户可以提交空数据，你可以试着不填写任何内容然后提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/QbyF9o4VH1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/QbyF9o4VH1.png!large)

这是数据库在抱怨数据不能为空，抛出异常了。接下来我们使用表单验证类来解决此问题，防止用户提交空数据，事实上，我们希望用户的回复至少有两个字符的长度：

_app/Http/Requests/ReplyRequest.php_

```
<?php

namespace App\Http\Requests;

class ReplyRequest extends Request
{
    public function rules()
    {
        return [
            'content' => 'required|min:2',
        ];
    }
}
```

再次尝试提交空数据，即可看到错误提示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/VFz4DEGTcV.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/VFz4DEGTcV.png!large)

## 5. 话题回复数

话题下每新增一条回复，此处应该 +1 ：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/xTzEkLPyyQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/xTzEkLPyyQ.png!large)

使用模型监控器能很容易实现此功能：

_app/Observers/ReplyObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Reply;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class ReplyObserver
{
    public function created(Reply $reply)
    {
        $reply->topic->increment('reply_count', 1);
    }
}
```

我们监控`created`事件，当 Elequont 模型数据成功创建时，`created`方法将会被调用。

再次尝试发布回复：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/awsxGidJLh.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/awsxGidJLh.png!large)

### 字段缓存

上面自增 1 是比较直接的做法，另一个比较严谨的做法是创建成功后计算本话题下评论总数，然后在对其`reply_count`字段进行赋值。这样做的好处多多，一般在做`xxx_count`此类总数缓存字段时，推荐使用此方法：

_app/Observers/ReplyObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Reply;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class ReplyObserver
{
    public function created(Reply $reply)
    {
        $reply->topic->reply_count = $reply->topic->replies->count();
        $reply->topic->save();
    }
}
```

可以看到现在评论数已将之前数据填充的评论计算在内：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/zrpaJWeoAN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/zrpaJWeoAN.png!large)

## 6. 处理 XSS 安全问题

我们在显示回复内容时，使用了 Blade 模板的`{!! !!}`『非转义打印』语法，在前面章节中我们已经提到过，这会是一个 XSS 安全威胁。我们可以尝试一下发布以下内容：

```
<script>alert('存在 XSS 安全威胁！')</script>
```

会出现弹框：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/Ec2lpjSUUR.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/Ec2lpjSUUR.gif!large)

我们将使用 HTMLPurifier 来修复此问题。与话题的类似地，我们将在模型监控器的`creating`事件中对`content`字段进行净化处理：

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

    public function creating(Reply $reply)
    {
        $reply->content = clean($reply->content, 'user_topic_body');
    }
}
```

话题回复的内容限定与话题的内容无异，因此我们使用同样的过滤规则 ——`user_topic_body`。

为了避免和上一次测试的混淆，我们重新选择一个话题，然后再次测试一次：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/2etWcmyFwZ.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/2etWcmyFwZ.gif!large)

可以看到问题已解决。至此评论发布功能开发完毕。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户可以发表评论"
```



