## 用户发布的话题

本章节我们在个人中心页面里，将当前用户的所有发布过的话题显示出来。

## 一、新增导航栏入口

为了更方便的进入『个人中心』，我们将在顶部导航栏新增个人中心的链接，并为下拉列表增加图标：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
            <div class="dropdown-menu" aria-labelledby="navbarDropdown">
              <a class="dropdown-item" href="{{ route('users.show', Auth::id()) }}">
                <i class="far fa-user mr-2"></i>
                个人中心
              </a>
              <div class="dropdown-divider"></div>
              <a class="dropdown-item" href="{{ route('users.edit', Auth::id()) }}">
                <i class="far fa-edit mr-2"></i>
                编辑资料
              </a>
              <div class="dropdown-divider"></div>
              <a class="dropdown-item" id="logout" href="#">
                <form action="{{ route('logout') }}" method="POST" onsubmit="return confirm('您确定要退出吗？');">
                  {{ csrf_field() }}
                  <button class="btn btn-block btn-danger" type="submit" name="button">退出</button>
                </form>
              </a>
            </div>
.
.
.
```

上面代码中，我们为下拉列表里的选项都加上了链接和图标，并且在退出登录的按钮那加了一个确认功能，防止用户误操作：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/9b6Sk8YoFg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/9b6Sk8YoFg.png!large)

## 二、新增模型关联

接下来我们在用户模型中新增与话题模型的关联：

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

    public function topics()
    {
        return $this->hasMany(Topic::class);
    }
}
```

用户与话题中间的关系是[一对多](https://learnku.com/docs/laravel/5.7/eloquent-relationships#one-to-many)的关系，一个用户拥有多个主题，在 Eloquent 中使用`hasMany()`方法进行关联。关联设置成功后，我们即可使用`$user->topics`来获取到用户发布的所有话题数据。

## 三、个人页面新增入口

接下来我们需要修改下个人页面的模板，将 『暂无数据～\_~』子串替换为新增『Ta 的话题』和『Ta 的回复』两个入口：

_resources/views/users/show.blade.php_

```
.
.
.
    {{-- 用户发布的内容 --}}
    <div class="card">
      <div class="card-body">
        <ul class="nav nav-tabs">
          <li class="nav-item"><a class="nav-link active bg-transparent" href="#">Ta 的话题</a></li>
          <li class="nav-item"><a class="nav-link" href="#">Ta 的回复</a></li>
        </ul>
        @include('users._topics', ['topics' => $user->topics()->recent()->paginate(5)])
      </div>
    </div>
.
.
.
```

话题的列表我们使用了子模板，接下来新建此文件，并写入以下内容：

_resources/views/users/\_topics.blade.php_

```
@if (count($topics))

  <ul class="list-group mt-4 border-0">
    @foreach ($topics as $topic)
      <li class="list-group-item pl-2 pr-2 border-right-0 border-left-0 @if($loop->first) border-top-0 @endif">
        <a href="{{ route('topics.show', $topic->id) }}">
          {{ $topic->title }}
        </a>
        <span class="meta float-right text-secondary">
          {{ $topic->reply_count }} 回复
          <span> ⋅ </span>
          {{ $topic->created_at->diffForHumans() }}
        </span>
      </li>
    @endforeach
  </ul>

@else
  <div class="empty-block">暂无数据 ~_~ </div>
@endif

{{-- 分页 --}}
<div class="mt-4 pt-1">
  {!! $topics->render() !!}
</div>
```

**代码高亮：**

```
@if($loop->first) border-top-0 @endif"
```

如果是第一个`list-group-item`的话，我们去掉上面的边。

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/seVlDo8Rqh.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/seVlDo8Rqh.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户话题列表"
```



