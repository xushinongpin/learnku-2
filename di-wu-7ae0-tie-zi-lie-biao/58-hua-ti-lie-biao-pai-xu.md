## 话题排序

虽然我们已经有话题列表，不过目前只有一种排序逻辑，本章节中，我们将让话题列表支持『最后回复』和『最新发布』排序：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/2Jm9eUOIIq.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/2Jm9eUOIIq.png!large)

我们可以通过 URI 传参`order`给控制器，控制器根据此参数来决定数据的读取逻辑。因为『分类下的话题列表』也会用到排序，并且是在不同的控制器中，所以在此处为了复用性考虑，我们将会把排序逻辑代码放置于 Topic 数据模型中。作为一个合格的程序员，编码时需时刻注意代码复用性。

接下来的步骤是：

* Topic 中编写排序逻辑；
* TopicsController 控制器中调用；
* CategoriesController 控制器中调用。

## 1. 编写排序逻辑

_app/Models/Topic.php_

```
.
.
.
class Topic extends Model
{
    .
    .
    .
    public function scopeWithOrder($query, $order)
    {
        // 不同的排序，使用不同的数据读取逻辑
        switch ($order) {
            case 'recent':
                $query->recent();
                break;

            default:
                $query->recentReplied();
                break;
        }
        // 预加载防止 N+1 问题
        return $query->with('user', 'category');
    }

    public function scopeRecentReplied($query)
    {
        // 当话题有新回复时，我们将编写逻辑来更新话题模型的 reply_count 属性，
        // 此时会自动触发框架对数据模型 updated_at 时间戳的更新
        return $query->orderBy('updated_at', 'desc');
    }

    public function scopeRecent($query)
    {
        // 按照创建时间排序
        return $query->orderBy('created_at', 'desc');
    }
}
```

这里我们使用了 Laravel[本地作用域](https://learnku.com/docs/laravel/5.7/eloquent#local-scopes)。本地作用域允许我们定义通用的约束集合以便在应用中复用。要定义这样的一个作用域，只需简单在对应 Eloquent 模型方法前加上一个 scope 前缀，作用域总是返回[查询构建器](https://learnku.com/docs/laravel/5.7/queries)。一旦定义了作用域，则可以在查询模型时调用作用域方法。在进行方法调用时不需要加上 scope 前缀。如以上代码中的`recent()`和`recentReplied()`。

## 2. 控制器中调用

接下来我们在话题控制器中链式调用定义的方法`withOrder()`：

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

    public function index(Request $request, Topic $topic)
    {
        $topics = $topic->withOrder($request->order)->paginate(20);
        return view('topics.index', compact('topics'));
    }
.
.
.
}
```

`$request->order`是获取 URI`http://larabbs.test/topics?order=recent`中的`order`参数。

---

接下来修改模板，我们需要为按钮添加链接还有选中状态：

_resources/views/topics/index.blade.php_

```
.
.
.

    <div class="card ">
      <div class="card-header bg-transparent">
        <ul class="nav nav-pills">
          <li class="nav-item">
            <a class="nav-link {{ active_class( ! if_query('order', 'recent')) }}" href="{{ Request::url() }}?order=default">
              最后回复
            </a>
          </li>
          <li class="nav-item">
            <a class="nav-link {{ active_class(if_query('order', 'recent')) }}" href="{{ Request::url() }}?order=recent">
              最新发布
            </a>
          </li>
        </ul>
      </div>

.
.
.
```

`Request::url()`获取的是当前请求的 URL，查看页面，通过 Laravel 开发者工具类查看读取列表数据的 SQL 请求，根据`updated_at`字段来排序：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/JzfkHQoHDL.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/JzfkHQoHDL.gif!large)

## 3. 分类话题列表排序

接下来我们在分类控制器中调用方法`withOrder()`：

_app/Http/Controllers/CategoriesController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Topic;
use App\Models\Category;

class CategoriesController extends Controller
{
    public function show(Category $category, Request $request, Topic $topic)
    {
        // 读取分类 ID 关联的话题，并按每 20 条分页
        $topics = $topic->withOrder($request->order)
                        ->where('category_id', $category->id)
                        ->paginate(20);

        // 传参变量话题和分类到模板中
        return view('topics.index', compact('topics', 'category'));
    }
}
```

测试分类下的排序：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/JcGISj9RLT.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/JcGISj9RLT.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "话题排序"
```



