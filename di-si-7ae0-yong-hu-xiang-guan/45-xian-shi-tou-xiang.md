## 显示头像

目前我们有两个地方用到用户头像，第一个是个人空间，第二个是顶部导航栏。

修改个人空间，将头像的`src`属性修改为`{{ $user->avatar }}`：

_resources/views/users/show.blade.php_

```
.
.
.
    <div class="card ">
      <img class="card-img-top" src="{{ $user->avatar }}" alt="{{ $user->name }}">
      <div class="card-body">
.
.
.
```

看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9hUxP7fA5Y.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9hUxP7fA5Y.png!large)

接下来修改顶部导航：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
            <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
              <img src="{{ Auth::user()->avatar }}" class="img-responsive img-circle" width="30px" height="30px">
              {{ Auth::user()->name }}
            </a>
.
.
.
```

查看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/ubzyUro5Lz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/ubzyUro5Lz.png!large)

## Git 代码版本控制

接着让我们将本次更改纳入版本控制中：

```
$ git add -A
$ git commit -m "显示头像"
```



