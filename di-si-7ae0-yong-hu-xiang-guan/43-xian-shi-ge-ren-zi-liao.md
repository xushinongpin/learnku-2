## 内容嵌套

上一节更新了个人简介，接下来我们将在个人中心里显示出来：

_resources/views/users/show.blade.php_

```
.
.
.
      <div class="card-body">
        <h5><strong>个人简介</strong></h5>
        <p>{{ $user->introduction }}</p>
        <hr>
        <h5><strong>注册于</strong></h5>
        <p>{{ $user->created_at->diffForHumans() }}</p>
      </div>
.
.
.
```

源码解读：

* $user-&gt;introduction 是调用上面我们新添加的字段；
* $user-&gt;created\_at-&gt;diffForHumans\(\) 时间戳友好的输出。

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/rh3Mf5A78n.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/rh3Mf5A78n.png!large)

页面中有两个问题：

1. 个人简介为空；
2. `7 hours ago `英文输出。

## 1. 个人简介为空

个人简介居然为空，我们明明在最后一次测试中填入内容了，并且也显示成功更新。让我们用数据库工具瞧一瞧是否有内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/3MJA0e9MKm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/3MJA0e9MKm.png!large)

居然是空的，所以很明显，刚刚数据并没有更新成功。

经过一番调试以后，原来是因为我们没有在`User.php`模型文件中，将`introduction`字段添加至`$fillable`属性中。`$fillable`属性的作用是防止用户随意修改模型数据，只有在此属性里定义的字段，才允许修改，否则更新时会被忽略。我们只需请按下图新增字段即可：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/qJ0ZdnTPev.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/qJ0ZdnTPev.png!large)

再次测试填写简介`Stay hungry, stay foolish.`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/1h17kJ2TzN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/1h17kJ2TzN.png!large)

成功显示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/PCSV4BAN2O.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/PCSV4BAN2O.png!large)

## 2. 中文显示友好时间戳

在 Laravel 中，时间戳`created_at`和`updated_at`作为模型属性被调用时，都会自动转换为`Carbon`对象，下面我们使用 Laravel 自带的`dd()`辅助函数验证一下：

_resources/views/users/show.blade.php_

```
.
.
.
      <div class="card-body">
        <h5><strong>个人简介</strong></h5>
        <p>{{ $user->introduction }}</p>
        <hr>
        <h5><strong>注册于</strong></h5>
        {{ dd($user->created_at) }}
        <p>{{ $user->created_at->diffForHumans() }}</p>
      </div>
.
.
.
```

打印出来的结果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/CKe3Fq4bOP.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/CKe3Fq4bOP.png!large)

[Carbon](https://github.com/briannesbitt/Carbon)是 PHP 知名的日期和时间操作扩展，Laravel 将其默认集成到了框架中。`diffForHumans`是`Carbon`对象提供的方法，默认情况是英文的，如果要使用中文时间提示，则需要对 Carbon 进行本地化设置。对 Carbon 进行本地化的设置很简单，只在`AppServiceProvider`中调用 Carbon 的`setLocale`方法即可，`AppServiceProvider`是框架的核心，在 Laravel 启动时，会最先加载该文件。

_app/Providers/AppServiceProvider.php_

```
.
.
.
    public function boot()
    {
        //

        \Carbon\Carbon::setLocale('zh');
    }
.
.
.
```

设置完成后，打开_resources/views/users/show.blade.php_去掉我们刚刚新增的`dd()`测试信息，刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/g2ZiMHAwqv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/g2ZiMHAwqv.png!large)

漂亮！

## Git 代码版本管理

接下来把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "显示个人资料"
```



