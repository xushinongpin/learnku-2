## 活跃用户

[![](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/kVdKbSWC9k.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/kVdKbSWC9k.png)

接下来我们将开发右边栏的『活跃用户』功能，此功能包括以下开发任务：

* 定义『活跃用户』算法
* 编写『活跃用户』逻辑代码
* 新建 Artisan 命令
* 数据读取和展示
* 设定计划任务

## 1. 活跃用户算法

算法我们将参考 Laravel China 社区：[关于「活跃用户」的算法](https://learnku.com/laravel/t/2902/algorithm-for-active-users)。

系统**每一个小时**计算一次，统计**最近 7 天**所有用户发的**帖子数**和**评论数**，用户每发一个帖子则得**4**分，每发一个回复得**1**分，计算出所有人的『得分』后再倒序，排名前八的用户将会显示在「活跃用户」列表里。

假设用户 A 在 7 天内发了 10 篇帖子，发了 5 条评论，则其得分为

```
10 * 4 + 5 * 1 = 45 
```

## 2. 编写逻辑代码

我们准备将『活跃用户』的逻辑代码放置于自定义的 Trait —— ActiveUserHelper 里，然后在 User 模型中加载此 Trait。使用 Trait 的加载方式既让相关的方法都存放于一处，便于查阅，另一方面，也保持了 User 模型的清爽。

新建此 Trait，并将算法相关的代码置于其中：

_app/Models/Traits/ActiveUserHelper.php_

```
<?php

namespace App\Models\Traits;

use App\Models\Topic;
use App\Models\Reply;
use Carbon\Carbon;
use Cache;
use DB;

trait ActiveUserHelper
{
    // 用于存放临时用户数据
    protected $users = [];       

    // 配置信息
    protected $topic_weight = 4; // 话题权重
    protected $reply_weight = 1; // 回复权重
    protected $pass_days = 7;    // 多少天内发表过内容
    protected $user_number = 6; // 取出来多少用户

    // 缓存相关配置
    protected $cache_key = 'larabbs_active_users';
    protected $cache_expire_in_minutes = 65;

    public function getActiveUsers()
    {
        // 尝试从缓存中取出 cache_key 对应的数据。如果能取到，便直接返回数据。
        // 否则运行匿名函数中的代码来取出活跃用户数据，返回的同时做了缓存。
        return Cache::remember($this->cache_key, $this->cache_expire_in_minutes, function(){
            return $this->calculateActiveUsers();
        });
    }

    public function calculateAndCacheActiveUsers()
    {
        // 取得活跃用户列表
        $active_users = $this->calculateActiveUsers();
        // 并加以缓存
        $this->cacheActiveUsers($active_users);
    }

    private function calculateActiveUsers()
    {
        $this->calculateTopicScore();
        $this->calculateReplyScore();

        // 数组按照得分排序
        $users = array_sort($this->users, function ($user) {
            return $user['score'];
        });

        // 我们需要的是倒序，高分靠前，第二个参数为保持数组的 KEY 不变
        $users = array_reverse($users, true);

        // 只获取我们想要的数量
        $users = array_slice($users, 0, $this->user_number, true);

        // 新建一个空集合
        $active_users = collect();

        foreach ($users as $user_id => $user) {
            // 找寻下是否可以找到用户
            $user = $this->find($user_id);

            // 如果数据库里有该用户的话
            if ($user) {

                // 将此用户实体放入集合的末尾
                $active_users->push($user);
            }
        }

        // 返回数据
        return $active_users;
    }

    private function calculateTopicScore()
    {
        // 从话题数据表里取出限定时间范围（$pass_days）内，有发表过话题的用户
        // 并且同时取出用户此段时间内发布话题的数量
        $topic_users = Topic::query()->select(DB::raw('user_id, count(*) as topic_count'))
                                     ->where('created_at', '>=', Carbon::now()->subDays($this->pass_days))
                                     ->groupBy('user_id')
                                     ->get();
        // 根据话题数量计算得分
        foreach ($topic_users as $value) {
            $this->users[$value->user_id]['score'] = $value->topic_count * $this->topic_weight;
        }
    }

    private function calculateReplyScore()
    {
        // 从回复数据表里取出限定时间范围（$pass_days）内，有发表过回复的用户
        // 并且同时取出用户此段时间内发布回复的数量
        $reply_users = Reply::query()->select(DB::raw('user_id, count(*) as reply_count'))
                                     ->where('created_at', '>=', Carbon::now()->subDays($this->pass_days))
                                     ->groupBy('user_id')
                                     ->get();
        // 根据回复数量计算得分
        foreach ($reply_users as $value) {
            $reply_score = $value->reply_count * $this->reply_weight;
            if (isset($this->users[$value->user_id])) {
                $this->users[$value->user_id]['score'] += $reply_score;
            } else {
                $this->users[$value->user_id]['score'] = $reply_score;
            }
        }
    }

    private function cacheActiveUsers($active_users)
    {
        // 将数据放入缓存中
        Cache::put($this->cache_key, $active_users, $this->cache_expire_in_minutes);
    }
}
```

在 Web 项目中，我们会经常使用到缓存系统。在合理使用的情况下，缓存系统很多时候能为程序带来巨大的性能提升。Laravel 给多种缓存系统提供丰富而统一的 API，缓存配置存放于`config/cache.php`。Laravel 支持当前流行的缓存系统，如非常棒的 Memcached 和 Redis。

在本教程中，我们将使用 Redis 作为我们的缓存驱动器。Redis 是内存缓存系统，将数据存放于内存中，读取速度飞快，是其他存储方式遥不可及的。不过因为内存大小有限，我们一般只用来存放『不经常修改，但是经常被读取』的数据。

在『活跃用户』的场景里，我们将使用计划任务，每一个小时计算一次活跃用户数据，计算成功后将数据存放于缓存中，页面展示需要此数据时，即可高速读取。

要使用 Redis 作为缓存驱动，首先需要在`.env`文件中将`CACHE_DRIVER`环境变量的值改为：

_.env_

```
.
.
.
CACHE_DRIVER=redis
.
.
.
```

算法的讲解，代码里有注释，这里便不再做过多讲解。下面在 User 模型中使用 Trait：

_app/Models/User.php_

```
<?php
.
.
.
class User extends Authenticatable implements MustVerifyEmailContract
{
    use Traits\ActiveUserHelper;
    .
    .
    .
}
```

## 3. 新建 Artisan 命令

使用以下生成命令类：

```
$ php artisan make:command CalculateActiveUser --command=larabbs:calculate-active-user
```

参数`--command`是指定 Artisan 调用的命令，一般情况下，我们推荐为命令加上命名空间，如本项目的`larabbs:`。

打开生成的 CalculateActiveUser 命令类文件，填入以下内容：

_app/Console/Commands/CalculateActiveUser.php_

```
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\User;

class CalculateActiveUser extends Command
{
    // 供我们调用命令
    protected $signature = 'larabbs:calculate-active-user';

    // 命令的描述
    protected $description = '生成活跃用户';

    // 最终执行的方法
    public function handle(User $user)
    {
        // 在命令行打印一行信息
        $this->info("开始计算...");

        $user->calculateAndCacheActiveUsers();

        $this->info("成功生成！");
    }
}
```

此刻使用`php artisan list`即可看到我们的命令行：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/pcdlpBaq6H.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/pcdlpBaq6H.png)

执行此命令生成『活跃用户』数据：

```
$ php artisan larabbs:calculate-active-user
```

修改下 TopicsController 中的`index()`方法，我们尝试打印从缓存里读取出来的数据（注意顶部 User 的引入）：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.
use App\Models\User;

class TopicsController extends Controller
{
    .
    .
    .
    public function index(Request $request, Topic $topic, User $user)
    {
        $topics = $topic->withOrder($request->order)->paginate(20);
        $active_users = $user->getActiveUsers();
        dd($active_users);
        return view('topics.index', compact('topics', 'active_users'));
    }
    .
    .
    .
}
```

首先顶部导入 User 类，并作为`index()`的第三个参数传参，这里这是只是一个便捷的`$user = new User`写法而已。

访问话题列表页面，即可看到我们的『活跃用户』集合：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/SC6CEjKmXt.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/SC6CEjKmXt.png!large)

测试完成后请将`dd($active_users);`删除。

## 4. 页面展示

我们已经可以读取到数据，接下来我们在左边栏中把数据显示出来。

修改右边栏模板，替换为以下：

_resources/views/topics/\_sidebar.blade.php_

```
<
div 
class
=
"card "
>
<
div 
class
=
"card-body"
>
<
a href
=
"{{ route('topics.create') }}"
class
=
"btn btn-success btn-block"
 aria
-
label
=
"Left Align"
>
<
i 
class
=
"fas fa-pencil-alt mr-2"
>
<
/
i
>
 新建帖子
    
<
/
a
>
<
/
div
>
<
/
div
>


@
if
(
count
(
$active_users
)
)
<
div 
class
=
"card mt-4"
>
<
div 
class
=
"card-body active-users pt-2"
>
<
div 
class
=
"text-center mt-1 mb-0 text-muted"
>
活跃用户
<
/
div
>
<
hr 
class
=
"mt-2"
>

      @
foreach
(
$active_users
as
$active_user
)
<
a 
class
=
"media mt-2"
 href
=
"{{ route('users.show', 
$active_user
-
>
id
) }}"
>
<
div 
class
=
"media-left media-middle mr-2 ml-1"
>
<
img src
=
"{{ 
$active_user
-
>
avatar
 }}"
 width
=
"24px"
 height
=
"24px"
class
=
"media-object"
>
<
/
div
>
<
div 
class
=
"media-body"
>
<
small 
class
=
"media-heading text-secondary"
>
{
{
$active_user
-
>
name
}
}
<
/
small
>
<
/
div
>
<
/
a
>

      @
endforeach
<
/
div
>
<
/
div
>

@
endif
```

刷新页面即可看到我们的活跃用户区块：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/SiVzp5UoVw.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/SiVzp5UoVw.png!large)

## 5. 计划任务

在过去，开发者必须为每个需要调度的任务生成单独的 Cron 项目。然而令人头疼的是任务调度不受版本控制，并且需要 SSH 到服务器上来增加 Cron 条目。

Laravel 命令调度器允许你在 Laravel 中对命令调度进行清晰流畅的定义，并且仅需要在服务器上增加一条 Cron 项目即可。调度在在`app/Console/Kernel.php`文件的`schedule`方法中定义。在该方法内包含了一个简单的例子，你可以随意增加调度到 Schedule 对象中。

使用调度器时，我们需要修改系统的 Cron 计划任务配置信息，运行以下命令：

```
$ export 
EDITOR
=
vi 
&
&
 crontab 
-
e
```

复制下面这一行：

```
*
*
*
*
*
 php 
/
home
/
vagrant
/
Code
/
larabbs
/
artisan schedule
:
run 
>
>
/
dev
/
null
2
>
&
1
```

此时进入 VI 编辑器界面：

1. 按大写的 G （或者按方向键）将光标移动到最底端；
2. 然后按键盘上的 『小写 o 键』进入 INSERT 模式；
3. 黏贴上面这一行；
4. 黏贴成功后按下键盘左上角的『ESC 键』进入 VI 的命令模式；
5. 键盘输入
   `:wq`
   并敲击回车键保存退出。

整个操作过程的动图如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/BkWsWCILbT.gif "file")](https://iocaffcdn.phphub.org/uploads/images/201710/24/1/BkWsWCILbT.gif)

系统的 Cron 已经设定好了，现在 Cron 软件将会每分钟调用一次 Laravel 命令调度器，当`schedule:run`命令执行时， Laravel 会评估你的计划任务并运行预定任务。

接下来将我们注册调度任务即可：

_app/Console/Kernel.php_

```
<
?php
.
.
.
class
Kernel
extends
ConsoleKernel
{
.
.
.
protected
function
schedule
(
Schedule 
$schedule
)
{
// $schedule-
>
command('inspire')
//          -
>
hourly();
// 一小时执行一次『活跃用户』数据生成的命令
$schedule
-
>
command
(
'larabbs:calculate-active-user'
)
-
>
hourly
(
)
;
}
.
.
.
}
```

## 6. 缓存效果示例

清空我们的缓存：

```
$ php artisan cache
:
clear
```

调出 Laravel 开发者工具类查看页面 SQL 请求数，可以看到没有缓存时，页面需要请求 18 条 SQL ，有了缓存只需要 5 条：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ixOATuzlJi.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/ixOATuzlJi.gif!large)

## 7. 分类话题列表

我们的分类话题列表页使用侧边栏，我们需要在分类控制器中将『活跃用户』数据传入模板中：

_app/Http/Controllers/CategoriesController.php_

```
<
?php
.
.
.
use
App
\
Models
\
User
;
class
CategoriesController
extends
Controller
{
public
function
show
(
Category 
$category
,
 Request 
$request
,
 Topic 
$topic
,
 User 
$user
)
{
// 读取分类 ID 关联的话题，并按每 20 条分页
$topics
=
$topic
-
>
withOrder
(
$request
-
>
order
)
-
>
where
(
'category_id'
,
$category
-
>
id
)
-
>
paginate
(
20
)
;
// 活跃用户列表
$active_users
=
$user
-
>
getActiveUsers
(
)
;
// 传参变量话题和分类到模板中
return
view
(
'topics.index'
,
compact
(
'topics'
,
'category'
,
'active_users'
)
)
;
}
}
```

注意类文件顶部 User 模型类的引入。现在访问话题分类页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/EJb0LB56vu.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/EJb0LB56vu.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add 
-
A

$ git commit 
-
m 
"活跃用户"
```



