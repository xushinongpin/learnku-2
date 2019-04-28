## 新建话题

接下来我们开发帖子发布功能，允许注册用户发布帖子，发布完成后，跳转到帖子详情页面。

## 新增入口

代码生成器已经为我们生成了新建话题的路由`topics.create`，我们需要在右边导航栏和顶部导航栏新增发帖入口：

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
          <li class="nav-item dropdown">
.
.
.
```

_resources/views/topics/\_sidebar.blade.php_

```
<div class="card ">
  <div class="card-body">
    <a href="{{ route('topics.create') }}" class="btn btn-success btn-block" aria-label="Left Align">
      <i class="fas fa-pencil-alt mr-2"></i>  新建帖子
    </a>
  </div>
</div>
```

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/15/1/uZwKNuWBhC.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/15/1/uZwKNuWBhC.png)

现在我们可以很方便的通过顶部导航栏进入话题发布页面。

## 数据模型

开始前，我们需要准备好话题数据模型 Topic 里的`$fillable`属性。代码生成器在生成模型时将所有的字段都罗列出来，这是很危险的，因为`$fillable`属性允许用户直接对数据进行修改，在每一次开发数据模型的 CRUD 功能时，我们都要慎重地对`$fillable`属性进行定制。给自己提一个问题：『哪些字段我们将不允许用户直接修改？』。在我们当前的情况下以下字段将禁止用户修改：

* user\_id —— 文章的作者，我们不希望文章的作者可以被随便指派；
* last\_reply\_user\_id —— 最后回复的用户 ID，将由程序来维护；
* order —— 文章排序，将会是管理员专属的功能；
* reply\_count —— 回复数量，程序维护；
* view\_count —— 查看数量，程序维护；

我们把以上字段从 Topic 数据模型的`$fillable`属性中移除：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
    protected $fillable = [
        'title', 'body', 'category_id', 'excerpt', 'slug'
    ];

    .
    .
    .
}
```

## 控制器

代码生成器已经帮我们生成了路由，请自行打开`routes/web.php`文件进行确认。我们将允许用户在发帖时选择分类，因此接下来我们将修改控制器的`create()`方法，将所有的分类读取赋值给变量`$categories`，并传入模板中：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.
use App\Models\Category;

class TopicsController extends Controller
{
    .
    .
    .
    public function create(Topic $topic)
    {
        $categories = Category::all();
        return view('topics.create_and_edit', compact('topic', 'categories'));
    }
    .
    .
    .
}
```

> 请注意在顶部引入`Category`。

接下来修改模板：

_resources/views/topics/create\_and\_edit.blade.php_

```
@extends('layouts.app')

@section('content')

  <div class="container">
    <div class="col-md-10 offset-md-1">
      <div class="card ">

        <div class="card-body">
          <h2 class="">
            <i class="far fa-edit"></i>
            @if($topic->id)
            编辑话题
            @else
            新建话题
            @endif
          </h2>

          <hr>

          @if($topic->id)
            <form action="{{ route('topics.update', $topic->id) }}" method="POST" accept-charset="UTF-8">
              <input type="hidden" name="_method" value="PUT">
          @else
            <form action="{{ route('topics.store') }}" method="POST" accept-charset="UTF-8">
          @endif

              <input type="hidden" name="_token" value="{{ csrf_token() }}">

              @include('shared._error')

              <div class="form-group">
                <input class="form-control" type="text" name="title" value="{{ old('title', $topic->title ) }}" placeholder="请填写标题" required />
              </div>

              <div class="form-group">
                <select class="form-control" name="category_id" required>
                  <option value="" hidden disabled selected>请选择分类</option>
                  @foreach ($categories as $value)
                  <option value="{{ $value->id }}">{{ $value->name }}</option>
                  @endforeach
                </select>
              </div>

              <div class="form-group">
                <textarea name="body" class="form-control" id="editor" rows="6" placeholder="请填入至少三个字符的内容。" required>{{ old('body', $topic->body ) }}</textarea>
              </div>

              <div class="well well-sm">
                <button type="submit" class="btn btn-primary"><i class="far fa-save mr-2" aria-hidden="true"></i> 保存</button>
              </div>
            </form>
        </div>
      </div>
    </div>
  </div>

@endsection
```

> 请注意此模板为创建和编辑帖子时共用的模板。

此时刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/rMhatl9iiL.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/rMhatl9iiL.png!large)

## 提交表单

随便写点内容，然后测试下提交表单，会出现以下报错：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/wEjuC4pcqR.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/wEjuC4pcqR.png!large)

这是因为我们入库的数据还未指定`user_id`字段值。`user_id`是用来记录话题作者对应的用户 ID，当用户提交文章时，我们将在控制器中，把当前登录的用户 ID 赋值给`user_id`：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.

use Auth;

class TopicsController extends Controller
{
    .
    .
    .

    public function store(TopicRequest $request, Topic $topic)
    {
        $topic->fill($request->all());
        $topic->user_id = Auth::id();
        $topic->save();

        return redirect()->route('topics.show', $topic->id)->with('success', '帖子创建成功！');
    }
.
.
.
}
```

代码解析：

* 因为要使用到
  `Auth`
  类，所以需在文件顶部进行加载；
* `store()`
  方法的第二个参数，会创建一个空白的 $topic 实例；
* `$request-`
  `>`
  `all()`
  获取所有用户的请求数据数组，如
  `['title' =`
  `>`
  ` '标题', 'body' =`
  `>`
  ` '内容', ... ]`
  ；
* `$topic-`
  `>`
  `fill($request-`
  `>`
  `all());`
  **fill**
  方法会将传参的键值数组填充到模型的属性中，如以上数组，
  `$topic-`
  `>`
  `title`
  的值为
  `标题`
  ；
* `Auth::id()`
  获取到的是当前登录的 ID；
* `$topic-`
  `>`
  `save()`
  保存到数据库中。

现在当我们再次提交即可看见创建成功的页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/lo7qL2WirK.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/lo7qL2WirK.png!large)

## 模型观察器

`excerpt`字段存储的是话题的摘录，将作为文章页面的`description`元标签使用，有利于 SEO 搜索引擎优化。摘录由文章内容中自动生成，生成的时机是在话题数据存入数据库之前。我们将使用 Eloquent 的[观察器](https://learnku.com/docs/laravel/5.7/eloquent#observers)来实现此功能。

Eloquent 模型会触发许多事件（Event），我们可以对模型的生命周期内多个时间点进行监控： creating, created, updating, updated, saving, saved, deleting, deleted, restoring, restored。事件让你每当有特定的模型类在数据库保存或更新时，执行代码。当一个新模型被初次保存将会触发`creating`以及`created`事件。如果一个模型已经存在于数据库且调用了`save`方法，将会触发`updating`和`updated`事件。在这两种情况下都会触发`saving`和`saved`事件。

Eloquent 观察器允许我们对给定模型中进行事件监控，观察者类里的方法名对应 Eloquent 想监听的事件。每种方法接收`model`作为其唯一的参数。代码生成器已经为我们生成了一个观察器文件，并在`AppServiceProvider`中注册。接下来我们要定制此观察器，在 Topic 模型保存时触发的`saving`事件中，对`excerpt`字段进行赋值：

_app/Observers/TopicObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Topic;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class TopicObserver
{
    public function saving(Topic $topic)
    {
        $topic->excerpt = make_excerpt($topic->body);
    }
}
```

`make_excerpt()`是我们自定义的辅助方法，我们需要在`helpers.php`文件中添加：

_app/helpers.php_

```
.
.
.

function make_excerpt($value, $length = 200)
{
    $excerpt = trim(preg_replace('/\r\n|\r|\n+/', ' ', strip_tags($value)));
    return str_limit($excerpt, $length);
}
```

## 表单验证类

接下来我们处理下表单验证规则，代码生成器已经为我们生成了`TopicRequest`表单验证类，并且自动在控制器方法中注入，我们不需要修改：

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

    public function store(TopicRequest $request, Topic $topic)
    {
        $topic->fill($request->all());
        $topic->user_id = Auth::id();
        $topic->save();

        return redirect()->route('topics.show', $topic->id)->with('success', '帖子创建成功！');
    }
    .
    .
    .
}
```

表单验证类里也有提供了基本的代码，我们需修改下验证规则和错误提醒：

_app/Http/Requests/TopicRequest.php_

```
<?php

namespace App\Http\Requests;

class TopicRequest extends Request
{
    public function rules()
    {
        switch($this->method())
        {
            // CREATE
            case 'POST':
            // UPDATE
            case 'PUT':
            case 'PATCH':
            {
                return [
                    'title'       => 'required|min:2',
                    'body'        => 'required|min:3',
                    'category_id' => 'required|numeric',
                ];
            }
            case 'GET':
            case 'DELETE':
            default:
            {
                return [];
            };
        }
    }

    public function messages()
    {
        return [
            'title.min' => '标题必须至少两个字符',
            'body.min' => '文章内容必须至少三个字符',
        ];
    }
}
```

在上面的写法中， 表单方法`POST`,`PUT`,`PATCH`使用的是相同的一套验证规则。

测试一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/HbkYwwbvbx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/HbkYwwbvbx.png!large)

## 新建帖子权限

我们还需要限制未登录用户发帖，代码生成器已经为我们生成了这块逻辑，这里我们检查确认即可：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.
class TopicsController extends Controller
{
    public function __construct()
    {

        $this->middleware('auth', ['except' => ['index', 'show']]);
    }

    .
    .
    .
}
```

`'except' => ['index', 'show']`—— 对除了`index()`和`show()`以外的方法使用`auth`中间件进行认证。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户可以创建话题"
```



