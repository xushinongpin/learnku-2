## 说明

上一章节中，我们往数据库里填充了 10 个用户和 100 条话题数据，本章节中我们将开发帖子列表页面，为这些话题数据提供访问的入口。

## 模型关联

开始之前，我们需要对 Topic 数据模型进行修改，新增`category`和`user`的模型关联：

* `category`
  —— 一个话题属于一个分类；
* `user`
  —— 一个话题拥有一个作者。

这两个关联都属于[一对一](https://learnku.com/docs/laravel/5.7/eloquent-relationships#one-to-one)对应关系，故我们使用`belongsTo()`方法来实现，代码如下：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
    protected $fillable = [
        'title', 'body', 'user_id', 'category_id', 'reply_count',
        'view_count', 'last_reply_user_id', 'order', 'excerpt', 'slug',
    ];

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

有了以上的关联设定，后面开发中我们可以很方便地通过`$topic->category`、`$topic->user`来获取到话题对应的分类和作者。

## 页面嵌套

接下来开始修改话题列表页面：

_resources/views/topics/index.blade.php_

```
@extends('layouts.app')

@section('title', '话题列表')

@section('content')

<div class="row mb-5">
  <div class="col-lg-9 col-md-9 topic-list">
    <div class="card ">

      <div class="card-header bg-transparent">
        <ul class="nav nav-pills">
          <li class="nav-item"><a class="nav-link active" href="#">最后回复</a></li>
          <li class="nav-item"><a class="nav-link" href="#">最新发布</a></li>
        </ul>
      </div>

      <div class="card-body">
        {{-- 话题列表 --}}
        @include('topics._topic_list', ['topics' => $topics])
        {{-- 分页 --}}
        <div class="mt-5">
          {!! $topics->appends(Request::except('page'))->render() !!}
        </div>
      </div>
    </div>
  </div>

  <div class="col-lg-3 col-md-3 sidebar">
    @include('topics._sidebar')
  </div>
</div>

@endsection
```

后面章节中，我们将会对列表增加排序功能，排序功能使用了 URL 传参来实现，这里使用分页中`appends()`方法可以使 URI 中的请求参数得到继承。

为了方便管理，话题列表被放置于子模板中：

_resources/views/topics/\_topic\_list.blade.php_

```
@if (count($topics))
  <ul class="list-unstyled">
    @foreach ($topics as $topic)
      <li class="media">
        <div class="media-left">
          <a href="{{ route('users.show', [$topic->user_id]) }}">
            <img class="media-object img-thumbnail mr-3" style="width: 52px; height: 52px;" src="{{ $topic->user->avatar }}" title="{{ $topic->user->name }}">
          </a>
        </div>

        <div class="media-body">

          <div class="media-heading mt-0 mb-1">
            <a href="{{ route('topics.show', [$topic->id]) }}" title="{{ $topic->title }}">
              {{ $topic->title }}
            </a>
            <a class="float-right" href="{{ route('topics.show', [$topic->id]) }}">
              <span class="badge badge-secondary badge-pill"> {{ $topic->reply_count }} </span>
            </a>
          </div>

          <small class="media-body meta text-secondary">

            <a class="text-secondary" href="#" title="{{ $topic->category->name }}">
              <i class="far fa-folder"></i>
              {{ $topic->category->name }}
            </a>

            <span> • </span>
            <a class="text-secondary" href="{{ route('users.show', [$topic->user_id]) }}" title="{{ $topic->user->name }}">
              <i class="far fa-user"></i>
              {{ $topic->user->name }}
            </a>
            <span> • </span>
            <i class="far fa-clock"></i>
            <span class="timeago" title="最后活跃于：{{ $topic->updated_at }}">{{ $topic->updated_at->diffForHumans() }}</span>
          </small>

        </div>
      </li>

      @if ( ! $loop->last)
        <hr>
      @endif

    @endforeach
  </ul>

@else
  <div class="empty-block">暂无数据 ~_~ </div>
@endif
```

> 注：注意上面多了一些类似写法`<i class="far fa-clock"></i>`，这是我们载入的 Font Awesome 字体图标库的写法，更多图标请[前往其官方文档](https://fontawesome.com/icons)。

右边栏：

_resources/views/topics/\_sidebar.blade.php_

```
<div class="card ">
  <div class="card-body">
    右边导航栏
  </div>
</div>
```

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/TzhJUAQVG4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/TzhJUAQVG4.png!large)

## 样式优化

基于 Bootstrap 默认的样式，我们只需要做下微调：

_resources/sass/app.scss_

```
.
.
.

/* Topic Index Page */
.topics-index-page {
  .topic-list {
    .nav>li>a {
      position: relative;
      display: block;
      padding: 5px 14px;
      font-size: 0.9em;
    }

    a {
      color: #444444;
    }

    .meta {
      font-size: 0.9em;
      color: #b3b3b3;

      a {
        color: #b3b3b3;
      }
    }

    .badge {
      background-color: #d8d8d8;
    }

    hr {
      margin-top: 12px;
      margin-bottom: 12px;
      border: 0;
      border-top: 1px solid #dcebf5;
    }
  }
}
```

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/8aJbwo2R18.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/8aJbwo2R18.png!large)

另外拉到最底部，我们可以发现页脚和内容黏在一起：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Z4KajkydiZ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Z4KajkydiZ.png!large)

再次对样式进行优化：

_resources/sass/app.scss_

```
.
.
.

/* Add container and footer space */
#app > div.container {
  margin-bottom: 100px;
}
```

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/xsmG8HwTp4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/xsmG8HwTp4.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "话题列表页面"
```



