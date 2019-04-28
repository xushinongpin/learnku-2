## 回复列表

本小结我们将构建回复列表，将所有的回复数据显示出来。总共有两个地方会用到回复数据：

1. 话题详情页下面的评论列表，是用户对此话题的评论；
2. 用户个人中心里的评论列表，罗列所有用户发表过的评论。

## 模板清理

代码生成器为我们生成了`resources/views/replies`文件夹以及文件夹下的三个模板文件，我们将用不到这些文件，开始之前我们先清理掉这些模板：

```
$ rm -rf resources/views/replies/
```

## 话题回复列表

接下来我们先构建话题页面下的用户回复列表：

_resources/views/topics/show.blade.php_

```
@extends('layouts.app')

@section('title', $topic->title)
@section('description', $topic->excerpt)

@section('content')

  <div class="row">

    <div class="col-lg-3 col-md-3 hidden-sm hidden-xs author-info">
      <div class="card ">
        <div class="card-body">
          <div class="text-center">
            作者：{{ $topic->user->name }}
          </div>
          <hr>
          <div class="media">
            <div align="center">
              <a href="{{ route('users.show', $topic->user->id) }}">
                <img class="thumbnail img-fluid" src="{{ $topic->user->avatar }}" width="300px" height="300px">
              </a>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="col-lg-9 col-md-9 col-sm-12 col-xs-12 topic-content">
      <div class="card">
        <div class="card-body">
          <h1 class="text-center mt-3 mb-3">
            {{ $topic->title }}
          </h1>

          <div class="article-meta text-center text-secondary">
            {{ $topic->created_at->diffForHumans() }}
            ⋅
            <i class="far fa-comment"></i>
            {{ $topic->reply_count }}
          </div>

          <div class="topic-body mt-4 mb-4">
            {!! $topic->body !!}
          </div>

          @can('update', $topic)
            <div class="operate">
              <hr>
              <a href="{{ route('topics.edit', $topic->id) }}" class="btn btn-outline-secondary btn-sm" role="button">
                <i class="far fa-edit"></i> 编辑
              </a>
              <form action="{{ route('topics.destroy', $topic->id) }}" method="post"
                    style="display: inline-block;"
                    onsubmit="return confirm('您确定要删除吗？');">
                {{ csrf_field() }}
                {{ method_field('DELETE') }}
                <button type="submit" class="btn btn-outline-secondary btn-sm">
                  <i class="far fa-trash-alt"></i> 删除
                </button>
              </form>
            </div>
          @endcan

        </div>
      </div>

      {{-- 用户回复列表 --}}
      <div class="card topic-reply mt-4">
          <div class="card-body">
              @include('topics._reply_box', ['topic' => $topic])
              @include('topics._reply_list', ['replies' => $topic->replies()->with('user')->get()])
          </div>
      </div>

    </div>
  </div>
@stop
```

我们只增加了尾部`{{-- 用户回复列表 --}}`这一段，注意读取回复列表时需使用懒加载来避免 N+1 问题。接下来说说我们新增的两个子模板：

* `_reply_box`
  回复框；
* `_reply_list`
  用户回复列表。

`_reply_box`子模板我们将在开发『创建回复』时进行处理，此时我们只需新建空白文件即可：

```
$ touch resources/views/topics/_reply_box.blade.php
```

接下来编写『回复列表子模板』的代码：

_resources/views/topics/\_reply\_list.blade.php_

```
<ul class="list-unstyled">
  @foreach ($replies as $index => $reply)
    <li class=" media" name="reply{{ $reply->id }}" id="reply{{ $reply->id }}">
      <div class="media-left">
        <a href="{{ route('users.show', [$reply->user_id]) }}">
          <img class="media-object img-thumbnail mr-3" alt="{{ $reply->user->name }}" src="{{ $reply->user->avatar }}" style="width:48px;height:48px;" />
        </a>
      </div>

      <div class="media-body">
        <div class="media-heading mt-0 mb-1 text-secondary">
          <a href="{{ route('users.show', [$reply->user_id]) }}" title="{{ $reply->user->name }}">
            {{ $reply->user->name }}
          </a>
          <span class="text-secondary"> • </span>
          <span class="meta text-secondary" title="{{ $reply->created_at }}">{{ $reply->created_at->diffForHumans() }}</span>

          {{-- 回复删除按钮 --}}
          <span class="meta float-right ">
            <a title="删除回复">
              <i class="far fa-trash-alt"></i>
            </a>
          </span>
        </div>
        <div class="reply-content text-secondary">
          {!! $reply->content !!}
        </div>
      </div>
    </li>

    @if ( ! $loop->last)
      <hr>
    @endif

  @endforeach
</ul>
```

注意此处回复内容显示时使用`{!! !!}`Blade 表达式，意味着非转义打印数据，这是一个安全隐患，我们将在『发布回复』功能的开发中处理此问题。

现在访问任意话题页面，即可看到我们的回复列表：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/QAXHEzb9qt.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/QAXHEzb9qt.png!large)

接下来优化下样式：

_resources/sass/app.scss_

```
.
.
.

/* 回复列表 */

.topic-reply {
    a {
        color: inherit;
    }

    .meta {
        font-size: .9em;
        color: #b3b3b3;
    }
}
```

刷新页面即可，链接的颜色温和一点了：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/1s4ujSOrFr.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/1s4ujSOrFr.png!large)

## 我的回复列表

下面我们开发『我的回复列表』，允许我们在用户的个人页面中，查看该用户发布过的所有回复数据。修改个人页面的模板：

_resources/views/users/show.blade.php_

```
.
.
.

    {{-- 用户发布的内容 --}}
    <div class="card ">
      <div class="card-body">
        <ul class="nav nav-tabs">
          <li class="nav-item">
            <a class="nav-link bg-transparent {{ active_class(if_query('tab', null)) }}" href="{{ route('users.show', $user->id) }}">
              Ta 的话题
            </a>
          </li>
          <li class="nav-item">
            <a class="nav-link bg-transparent {{ active_class(if_query('tab', 'replies')) }}" href="{{ route('users.show', [$user->id, 'tab' => 'replies']) }}">
              Ta 的回复
            </a>
          </li>
        </ul>
        @if (if_query('tab', 'replies'))
          @include('users._replies', ['replies' => $user->replies()->with('topic')->recent()->paginate(5)])
        @else
          @include('users._topics', ['topics' => $user->topics()->recent()->paginate(5)])
        @endif
      </div>
    </div>

    </div>
</div>
@stop
```

`recent()`方法在数据模型基类`app/Models/Model.php`中定义，并且使用了[本地作用域](https://learnku.com/docs/laravel/5.7/eloquent#local-scopes)的方式进行定义，我们的 Reply 模型，就如代码生成器所生成的数据模型一样，统一继承了此类方法：

```
public function scopeRecent($query)
{
    return $query->orderBy('id', 'desc');
}
```

接下来创建子模板：

_resources/views/users/\_replies.blade.php_

```
@if (count($replies))

  <ul class="list-group mt-4 border-0">
    @foreach ($replies as $reply)
      <li class="list-group-item pl-2 pr-2 border-right-0 border-left-0 @if($loop->first) border-top-0 @endif">
        <a href="{{ $reply->topic->link(['#reply' . $reply->id]) }}">
          {{ $reply->topic->title }}
        </a>

        <div class="reply-content text-secondary mt-2 mb-2">
          {!! $reply->content !!}
        </div>

        <div class="text-secondary" style="font-size:0.9em;">
          <i class="far fa-clock"></i> 回复于 {{ $reply->created_at->diffForHumans() }}
        </div>
      </li>
    @endforeach
  </ul>

@else
  <div class="empty-block">暂无数据 ~_~ </div>
@endif

{{-- 分页 --}}
<div class="mt-4 pt-1">
  {!! $replies->appends(Request::except('page'))->render() !!}
</div>
```

我们使用了 URL 中的`tab`请求参数对话题列表进行区分，分页中`appends()`方法可以使 URI 中的请求参数得到继承。

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/neckZE90OC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/25/1/neckZE90OC.png!large)

至此两个回复列表开发完毕。

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "回复列表"
```



