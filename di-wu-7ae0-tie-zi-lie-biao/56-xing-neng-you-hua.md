## 页面调优

此刻我们的页面存在很大的**性能隐患**，为了能更直观地看到问题，我们先安装 Laravel 开发者工具类 -[laravel-debugbar](https://github.com/barryvdh/laravel-debugbar)。

### 安装 Debugbar

使用 Composer 安装：

```
$ composer require "barryvdh/laravel-debugbar:~3.2" --dev
```

生成配置文件，存放位置`config/debugbar.php`：

```
$ php artisan vendor:publish --provider="Barryvdh\Debugbar\ServiceProvider"
```

打开`config/debugbar.php`，将`enabled`的值设置为：

```
'enabled' => env('APP_DEBUG', false),
```

修改完以后，Debugbar 分析器的启动状态将由`.env`文件中`APP_DEBUG`值决定。

刷新列表页面即可看到我们的开发者工具栏：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/jyKa8ncWzx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/jyKa8ncWzx.png!large)

### N +1 问题

如图点击以下按钮，可看到整个页面执行了 33 条查询语句，往下滚动可以看到很多请求都是重复的：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lN5bAAfYTo.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lN5bAAfYTo.gif!large)

Laravel 的默认分页是 15 条信息，如果我们在控制器中修改`paginate(30)`显示条目为 30 的话：

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
    public function index()
    {
        $topics = Topic::paginate(30);
        return view('topics.index', compact('topics'));
    }
    .
    .
    .
}
```

可以看到现在的 SQL 查询数量为 62，是之前的两倍：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cygQSAv2eq.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cygQSAv2eq.png!large)

> 提示：62 statements were executed, 60 of which were duplicated, 2 unique  
> 意为：总共有 62 条语句执行了，其中 60 条是重复的。

以上的问题就是 N+1 问题，不仅是 Laravel 中，所有的 ORM 关联数据读取中都存在此问题，新手很容易踩到坑。进而导致系统变慢，然后拖垮整个系统。

N+1 一般发生在关联数据的遍历时。在`resources/views/topics/_topic_list.blade.php`模板中，我们对`$topics`进行遍历，为了方便解说，我们将此文件里的代码精简为如下：

```
@if (count($topics))

    <ul class="media-list">
        @foreach ($topics as $topic)
            .
            .
            .
            {{ $topic->user->name }}
            .
            .
            .
            {{ $topic->category->name }}
            .
            .
            .
        @endforeach
    </ul>

@else
   <div class="empty-block">暂无数据 ~_~ </div>
@endif
```

为了读取`user`和`category`，每次的循环都要查一下`users`和`categories`表，在本例子中我们查询了 30 条话题数据，那么最终我需要执行的查询语句就是 30 \* 2 + 1 = 61 条语句。如果我第一次查询出来的是 N 条记录，那么最终需要执行的 SQL 语句就是 N+1 次。

## 如何解决 N + 1 问题？

我们可以通过 Eloquent 提供的[预加载功能](https://learnku.com/docs/laravel/5.7/eloquent-relationships#eager-loading)来解决此问题：

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
    public function index()
    {
        $topics = Topic::with('user', 'category')->paginate(30);
        return view('topics.index', compact('topics'));
    }
    .
    .
    .
}
```

方法`with()`提前加载了我们后面需要用到的关联属性`user`和`category`，并做了缓存。后面即使是在遍历数据时使用到这两个关联属性，数据已经被预加载并缓存，因此不会再产生多余的 SQL 查询：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/SKLfiVtrIJ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/SKLfiVtrIJ.png!large)

上图可以看到优化完成后，我们的 SQL 查询数量瞬间减少到只有 4 条，相应的，页面的响应时间也减少了三分之一。

## 最小化开发者工具类

我们可以通过以下方法将 Laravel 开发者工具类最小化，使其变得很不显眼，后续开发大家可以按需调整：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/TSmPlXwr43.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/TSmPlXwr43.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "修复 N+1 问题"
```



