## SEO 友好的 URL

释义的 URL 有助于搜索引擎优化（SEO），本章节我们将开发自动生成 SEO 友好 URL 的功能。当用户提交发布话题的表单时，程序将调用[百度翻译](http://api.fanyi.baidu.com/api/trans/product/apidoc)接口将话题标题翻译为英文，并储存于字段`slug`中。显示时候将 Slug 在 URL 中体现出来，假如话题标题为『Slug 翻译测试』的 URL 是：

> [http://larabbs.test/topics/119](http://larabbs.test/topics/119)

加入 Slug 后 SEO 友好的链接为：

> [http://larabbs.test/topics/119/slug-translation-test](http://larabbs.test/topics/119/slug-translation-test)

## 翻译处理器

首先，我们需将翻译的全部逻辑封装为一个类，并放置于`Handlers`文件夹中：

_app/Handlers/SlugTranslateHandler.php_

```
<?php

namespace App\Handlers;

use GuzzleHttp\Client;
use Overtrue\Pinyin\Pinyin;

class SlugTranslateHandler
{
    public function translate($text)
    {
        // 实例化 HTTP 客户端
        $http = new Client;

        // 初始化配置信息
        $api = 'http://api.fanyi.baidu.com/api/trans/vip/translate?';
        $appid = config('services.baidu_translate.appid');
        $key = config('services.baidu_translate.key');
        $salt = time();

        // 如果没有配置百度翻译，自动使用兼容的拼音方案
        if (empty($appid) || empty($key)) {
            return $this->pinyin($text);
        }

        // 根据文档，生成 sign
        // http://api.fanyi.baidu.com/api/trans/product/apidoc
        // appid+q+salt+密钥 的MD5值
        $sign = md5($appid. $text . $salt . $key);

        // 构建请求参数
        $query = http_build_query([
            "q"     =>  $text,
            "from"  => "zh",
            "to"    => "en",
            "appid" => $appid,
            "salt"  => $salt,
            "sign"  => $sign,
        ]);

        // 发送 HTTP Get 请求
        $response = $http->get($api.$query);

        $result = json_decode($response->getBody(), true);

        /**
        获取结果，如果请求成功，dd($result) 结果如下：

        array:3 [▼
            "from" => "zh"
            "to" => "en"
            "trans_result" => array:1 [▼
                0 => array:2 [▼
                    "src" => "XSS 安全漏洞"
                    "dst" => "XSS security vulnerability"
                ]
            ]
        ]

        **/

        // 尝试获取获取翻译结果
        if (isset($result['trans_result'][0]['dst'])) {
            return str_slug($result['trans_result'][0]['dst']);
        } else {
            // 如果百度翻译没有结果，使用拼音作为后备计划。
            return $this->pinyin($text);
        }
    }

    public function pinyin($text)
    {
        return str_slug(app(Pinyin::class)->permalink($text));
    }
}
```

在类实例化以后，我们只需要调用`translate()`方法即可得到翻译的结果。不过目前我们还需安装依赖的扩展包。

### 1. 安装依赖 Guzzle

[Guzzle](https://github.com/guzzle/guzzle)库是一套强大的 PHP HTTP 请求套件，我们使用 Guzzle 的 HTTP 客户端来请求[百度翻译](http://api.fanyi.baidu.com/api/trans/product/apidoc)接口。

使用 Composer 安装 Guzzle 类库：

```
$ composer require "guzzlehttp/guzzle:~6.3"
```

无需配置，安装完成后即可使用。我们已在 SlugTranslateHandler.php 顶部`use`引入使用。

### 2. 安装依赖 PinYin

[PinYin](https://github.com/overtrue/pinyin)是[安正超](https://learnku.com/users/76)开发的，基于[CC-CEDICT](http://cc-cedict.org/wiki/)词典的中文转拼音工具，是一套优质的汉字转拼音解决方案。我们使用 PinYin 来作为翻译的后备计划，当百度翻译 API 不可用时，程序会自动使用 PinYin 汉字转拼音方案来生成 Slug。

使用 Composer 安装 PinYin 类库：

```
$ composer require "overtrue/pinyin:~3.0"
```

同样的，我们已在 SlugTranslateHandler.php 顶部`use`引入使用。

### 3. 百度翻译 API 配置信息

当使用百度翻译 API 时，我们需要申请官方授权的`appid`和`key`。打开[百度翻译开放平台](http://api.fanyi.baidu.com/api/trans/product/index)，然后点击『申请接入』按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201810/26/1/25tGc8HvRg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201810/26/1/25tGc8HvRg.png!large)

百度翻译的审核并不严苛，选填的地方可以跳过不填，信息填写后提交即可生成授权`key`，此刻访问『管理控制台』即可看到`appid`和`key`：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/BqjZismtQP.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/19/1/BqjZismtQP.png)

接下来我们将`appid`和`key`集成到的项目中。一般情况下，我们将这种第三方服务授权认证信息存放于`services.php`里：

_config/services.php_

```
<?php

return [

    .
    .
    .

    'baidu_translate' => [
        'appid' => env('BAIDU_TRANSLATE_APPID'),
        'key'   => env('BAIDU_TRANSLATE_KEY'),
    ],

];
```

我们还希望不同环境下对配置信息有不同的设定，所以在此处我们通过`env()`方法来引用环境变量，接下来我们需要在本地环境变量中新增这两个值，请将这两个值替换为你申请的值：

_.env_

```
.
.
.

BAIDU_TRANSLATE_APPID=201703xxxxxxxxxxxxx
BAIDU_TRANSLATE_KEY=q0s6axxxxxxxxxxxxxxxxx
```

每当我们在`.env`中新增键值时，都必须在`.env.example`文件中新增相应的键：

_.env.example_

```
.
,
,

BAIDU_TRANSLATE_APPID=
BAIDU_TRANSLATE_KEY=
```

因为`.env`文件被我们排除 Git 跟踪（可以查看 .gitignore 文件），文件`.env.example`是作为项目环境变量的初始化文件而存在。当项目在新环境中安装时，只需要执行`cp .env.example .env`命令，并在`.env`填入对应的值，即可完成对项目环境变量的配置。

## 翻译调用

我们只需在入库前对 slug 字段进行赋值即可，我们可以方便地在 Eloquent 观察器中实现：

_app/Observers/TopicObserver.php_

```
<?php

namespace App\Observers;

use App\Models\Topic;
use App\Handlers\SlugTranslateHandler;

// creating, created, updating, updated, saving,
// saved,  deleting, deleted, restoring, restored

class TopicObserver
{
    public function saving(Topic $topic)
    {
        // XSS 过滤
        $topic->body = clean($topic->body, 'user_topic_body');

        // 生成话题摘录
        $topic->excerpt = make_excerpt($topic->body);

        // 如 slug 字段无内容，即使用翻译器对 title 进行翻译
        if ( ! $topic->slug) {
            $topic->slug = app(SlugTranslateHandler::class)->translate($topic->title);
        }
    }
}
```

`app()`允许我们使用[Laravel 服务容器](https://learnku.com/docs/laravel/5.7/container)，此处我们用来生成 SlugTranslateHandler 实例。

测试一下，成功发布一篇文章以后：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/f2H7B0BnTS.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/f2H7B0BnTS.png!large)

数据库查看我们最新发布的一条数据（没看到数据的同学 ctrl + r 刷新）：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/b5hYQTekkC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/b5hYQTekkC.png!large)

## 显示修改

现在我们已经能生成 Slug 了，接下来我们需要将其显示出来。路由规则有了变化，我们需要修改路由文件。为了方便调用和管理，我们将在 Topic 模型中新建方法`link()`来生成模型 URL，然后将所有调用了`route('topics.show', $topic->id)`的地方修改为`$topic->link()`。

### 1. 修改路由文件

在`routes/web.php`文件中，找到以下这一行：

```
Route::resource('topics', 'TopicsController', ['only' => ['index', 'show', 'create', 'store', 'update', 'edit', 'destroy']]);
```

修改为：

```
Route::resource('topics', 'TopicsController', ['only' => ['index', 'create', 'store', 'update', 'edit', 'destroy']]);
```

然后在`routes/web.php`文件的最后新增以下路由规则：

_`routes/web.php`_

```
.
.
.
Route::get('topics/{topic}/{slug?}', 'TopicsController@show')->name('topics.show');
```

URI 参数`topic`是[『隐性路由模型绑定』](https://learnku.com/docs/laravel/5.7/routing#%E9%9A%90%E5%BC%8F%E7%BB%91%E5%AE%9A)的提示，将会自动注入 ID 对应的话题实体。

URI 最后一个参数表达式`{slug?}`，`?`意味着参数可选，这是为了兼容我们数据库中 Slug 为空的话题数据。这种写法可以同时兼容以下两种链接：

* [http://larabbs.test/topics/115](http://larabbs.test/topics/115)
* [http://larabbs.test/topics/115/slug-translation-test](http://larabbs.test/topics/115/slug-translation-test)

### 2. 新建 link \(\) 方法：

_app/Models/Topic.php_

```
<?php

namespace App\Models;

class Topic extends Model
{
    .
    .
    .

    public function link($params = [])
    {
        return route('topics.show', array_merge([$this->id, $this->slug], $params));
    }
}
```

参数`$params`允许附加 URL 参数的设定。

### 3. 全局修改

使用编辑器全局搜索关键字`topics.show`：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/18/1/vW1hE8J7R9.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/18/1/vW1hE8J7R9.png)

然后手动一个个修改，主要由两种链接，一种是控制器里的跳转，例如：

```
return redirect()->route('topics.show', $topic->id)->with('success', '成功创建话题！');
```

修改为：

```
return redirect()->to($topic->link())->with('success', '成功创建话题！');
```

另一种是模板里的，例如：

```
<a href="{{ route('topics.show', [$topic->id]) }}" title="{{ $topic->title }}">
    {{ $topic->title }}
</a>
```

修改为：

```
<a href="{{ $topic->link() }}" title="{{ $topic->title }}">
    {{ $topic->title }}
</a>
```

所有地方修改完成后，打开[http://larabbs.test/topics](http://larabbs.test/topics)查看链接情况：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/nwm97blpOb.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/nwm97blpOb.gif!large)

## 强制跳转

当话题有 Slug 的时候，我们希望用户一直使用正确的、带着 Slug 的链接来访问。我们可以在控制器中对 Slug 进行判断，当条件允许的时候，我们将发送 301 永久重定向指令给浏览器，跳转到带 Slug 的链接：

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
    public function show(Request $request, Topic $topic)
    {
        // URL 矫正
        if ( ! empty($topic->slug) && $topic->slug != $request->slug) {
            return redirect($topic->link(), 301);
        }

        return view('topics.show', compact('topic'));
    }
    .
    .
    .
}
```

代码讲解：

1. 我们需要访问用户请求的路由参数 Slug，在
   `show()`
   方法中我们注入
   `$request`
   ；
2. `! empty($topic-`
   `>`
   `slug)`
   如果话题的 Slug 字段不为空；
3. `&`
   `&`
   ` $topic-`
   `>`
   `slug != $request-`
   `>`
   `slug`
   并且话题 Slug 不等于请求的路由参数 Slug；
4. `redirect($topic-`
   `>`
   `link(), 301)`
   301 永久重定向到正确的 URL 上。

测试一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/B0r646Osez.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/B0r646Osez.gif!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "翻译 URL 标示"
```



