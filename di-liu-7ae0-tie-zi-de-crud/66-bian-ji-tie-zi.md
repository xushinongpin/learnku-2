## 编辑帖子

接下来我们要实现此功能：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/Fmb8j4ifXK.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/Fmb8j4ifXK.png!large)

现在点击以上的编辑按钮，会出现以下报错：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/d38DmIh2U5.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/d38DmIh2U5.png!large)

那是因为我们的创建和编辑使用的是同一个模板文件`create_and_edit.blade.php`，在前面开发创建话题功能时，我们往控制器方法`create`传参了`$categories`变量，供用户选择话题所属的分类。接下来我们只需要在负责展示编辑页面的控制器方法`edit()`中传参`$categories`变量即可：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.
class TopicsController extends Controller
{
    .
    .
    .

    public function edit(Topic $topic)
    {
        $this->authorize('update', $topic);
        $categories = Category::all();
        return view('topics.create_and_edit', compact('topic', 'categories'));
    }
    .
    .
    .
}
```

再次刷新页面，页面成功显示。不过分类选择那并没有选中，我们需要修改一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/fVKvXt4c2X.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/fVKvXt4c2X.png!large)

打开模板文件，在分类选择那里加上逻辑判断，有两个地方需要判断，第一个 『请选择分类』那，如果是编辑情况下就去掉`selected`，因为编辑情况下，肯定已经做过分类的选择。第二个是哪个分类被选中的判断，遍历时只要与话题关联的`category_id`一致的话，即可视为选中：

_resources/views/topics/create\_and\_edit.blade.php_

```
.
.
.
              <div class="form-group">
                <select class="form-control" name="category_id" required>
                  <option value="" hidden disabled {{ $topic->id ? '' : 'selected' }}>请选择分类</option>
                    @foreach ($categories as $value)
                      <option value="{{ $value->id }}" {{ $topic->category_id == $value->id ? 'selected' : '' }}>
                        {{ $value->name }}
                      </option>
                    @endforeach
                </select>
              </div>
.
.
.
```

刷新页面后即可看到选中状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/XOhEaDklPV.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/XOhEaDklPV.png!large)

## 测试一下

接下来测试下文章修改功能，尝试修改点东西：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/GAUe1wMV04.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/GAUe1wMV04.png!large)

提交后可以看到修改成功的提示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/sHKywBThsz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/sHKywBThsz.png!large)

## 权限控制

我们将只允许作者对话题有编辑权限。代码生成器已经为我们准备好授权策略（Policy）类，接下来我们将对`update()`方法进行修改，只有当话题关联作者的 ID 等于当前登录用户 ID 时候才放行：

_app/Policies/TopicPolicy.php_

```
<?php

namespace App\Policies;

use App\Models\User;
use App\Models\Topic;

class TopicPolicy extends Policy
{
    public function update(User $user, Topic $topic)
    {
        return $topic->user_id == $user->id;
    }

    .
    .
    .
}
```

在授权策略的类方法里，返回`true`即允许访问，反之返回`false`为拒绝访问。

接下来是调用，同样的，代码生成器细心地在我们的 TopicsController 里做了授权策略的调用，我们需检查此文件中`authorize()`方法的调用，并修改下提示信息 ：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.

class TopicsController extends Controller
{
    .
    .
    .

    public function edit(Topic $topic)
    {
        $this->authorize('update', $topic);
        $categories = Category::all();
        return view('topics.create_and_edit', compact('topic', 'categories'));
    }

    public function update(TopicRequest $request, Topic $topic)
    {
        $this->authorize('update', $topic);
        $topic->update($request->all());

        return redirect()->route('topics.show', $topic->id)->with('success', '更新成功！');
    }

    .
    .
    .
}
```

我们可以从数据库里找一条数据，作者非 1 号用户：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/MwPVZ3XCjw.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/MwPVZ3XCjw.png!large)

复制此话题的 ID ，替换为 ——[http://larabbs.test/topics/100/edit](http://larabbs.test/topics/100/edit)进行访问，将会返回 403 未授权访问：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/hN8B8zRWJi.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/hN8B8zRWJi.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户可以编辑话题"
```



