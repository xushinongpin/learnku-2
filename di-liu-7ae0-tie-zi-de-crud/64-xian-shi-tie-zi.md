## 帖子详情页

现在我们已经能撰写文章，发布帖子了，接下来我们把发布的内容显示出来。

## 1. 路由和控制器

代码生成器已经为我们生成了路由和控制器的代码，我们看下`show()`方法：

_app/Http/Controllers/TopicsController.php_

```
.
.
.
class TopicsController extends Controller
{
    .
    .
    .
    public function show(Topic $topic)
    {
        return view('topics.show', compact('topic'));
    }
    .
    .
    .
}
```

注意此处使用 Laravel 的[『隐性路由模型绑定』](https://learnku.com/docs/laravel/5.7/routing#%E9%9A%90%E5%BC%8F%E7%BB%91%E5%AE%9A)功能，当请求[http://larabbs.test/topics/1](http://larabbs.test/topics/1)时，`$topic`变量会自动解析为 ID 为 1 的帖子对象。

## 2. 修改模板

接下来修改模板：

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
      <div class="card ">
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

          <div class="operate">
            <hr>
            <a href="{{ route('topics.edit', $topic->id) }}" class="btn btn-outline-secondary btn-sm" role="button">
              <i class="far fa-edit"></i> 编辑
            </a>
            <a href="#" class="btn btn-outline-secondary btn-sm" role="button">
              <i class="far fa-trash-alt"></i> 删除
            </a>
          </div>

        </div>
      </div>
    </div>
  </div>
@stop
```

话题字段`excerpt`是文章摘录，用作 SEO 页面描述使用，因此我们还需要修改主布局模板，新增`description`原标签：

_resources/views/layouts/app.blade.php_

```
.
.
.
  <title>@yield('title', 'LaraBBS') - Laravel 进阶教程</title>
  <meta name="description" content="@yield('description', 'LaraBBS 爱好者社区')" />
.
.
.
```

## 3. 测试一下

为了方便测试，我们将复制[这篇文章](https://learnku.com/articles/3405/why-is-php-the-best-language-it-is-and-will-be)内容到我们新建的文章中，顺便发现了一个样式问题：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/MLibBQXYtq.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/MLibBQXYtq.png!large)

快速修复下：

_resources/sass/app.scss_

```
.
.
.

.simditor-body img {
  max-width:100%;
}
```

再次尝试，可以看到样式已修复。写入标题，选择分类，复制黏贴内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/1tM8yN4CIt.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/1tM8yN4CIt.png!large)

点击保存：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/8p4vtuRgjk.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/8p4vtuRgjk.png!large)

发现话题内容的样式比较乱，接下来我们一起调整下样式。

## 4. 样式调整

除了页面结构的调整外，我们还有话题内容的样式需要调整。下面这份内容样式来自于[Laravel China 社区](https://laravel-china.org/)站点，我们直接使用即可。因为内容较多，我们将其放置于另一个文件中：

_resources/sass/\_topic\_body.scss_

```
.simditor-body, .topic-body {
  font-size: 15px;
  line-height: 1.3;
  overflow: hidden;
  line-height: 1.6;
  word-wrap: break-word;

  a {
    background: transparent;
  }

  a:active,
  a:hover {
    outline: 0;
  }

  ol li {
    margin: 8px 0;
  }

  pre[class*=language-] {
    margin: 1.2em 0 !important;
  }

  strong {
    font-weight: bold;
  }

  h1 {
    font-size: 2em;
    margin: 0.67em 0;
  }

  img {
    border: 0;
  }

  hr {
    -moz-box-sizing: content-box;
    box-sizing: content-box;
    height: 0;
  }

  table {
    border-collapse: collapse;
    border-spacing: 0;
  }

  td,
  th {
    padding: 0;
  }

  * {
    -moz-box-sizing: border-box;
    box-sizing: border-box;
  }

  a {
    text-decoration: none;
  }

  a:hover,
  a:focus,
  a:active {
    text-decoration: none;
  }

  hr {
    height: 0;
    margin: 15px 0;
    overflow: hidden;
    background: transparent;
    border: 0;
    border-bottom: 1px solid #ddd;
  }

  hr:before,
  hr:after {
    display: table;
    content: " ";
  }

  hr:after {
    clear: both;
  }

  blockquote {
    margin: 0;
  }

  ul,
  ol {
    padding: 0;
    margin-top: 0;
    margin-bottom: 0;
  }

  ol ol {
    list-style-type: lower-roman;
  }

  dd {
    margin-left: 0;
  }

  code,
  pre {
    font-family: monaco, Consolas, "Liberation Mono", Menlo, Courier, monospace;
    font-size: 1em;
  }

  pre {
    margin-top: 0;
    margin-bottom: 0;
    overflow: auto;
  }

  .topic-body>*:first-child {
    margin-top: 0 !important;
  }

  .topic-body>*:last-child {
    margin-bottom: 0 !important;
  }

  .anchor {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    display: block;
    padding-right: 6px;
    padding-left: 30px;
    margin-left: -30px;
  }

  .anchor:focus {
    outline: none;
  }

  h1,
  h2,
  h3,
  h4,
  h5,
  h6 {
    position: relative;
    margin-top: 1.0em;
    margin-bottom: 16px;
    line-height: 1.4;
  }

  h1 {
    padding-bottom: 0.3em;
    font-size: 2.25em;
    line-height: 1.2;
    border-bottom: 1px solid #eee;
  }

  h2 {
    padding-bottom: 0.3em;
    font-size: 1.3em;
    line-height: 1.225;
    border-bottom: 1px solid #eee;
  }

  h3 {
    font-size: 1.2em;
    line-height: 1.43;
  }

  h4 {
    font-size: 1.1em;
  }

  h5 {
    font-size: 1.0em;
  }

  h6 {
    font-size: 0.9em;
    color: #777;
  }

  p,
  blockquote,
  ul,
  ol,
  dl,
  table,
  pre {
    margin-top: 0;
    margin-bottom: 0px;
    line-height: 30px;
  }

  hr {
    border: 2px dashed #F0F4F6;
    border-bottom: 0px;
    margin: 18px auto;
    width: 50%;
  }

  ul,
  ol {
    padding-left: 2em;
    padding: 10px 20px 10px 30px;
    color: #7d8688;
  }

  ol ol,
  ol ul {
    margin-top: 0;
    margin-bottom: 0;
  }

  li>p {
    margin-top: 6px;
  }

  dl {
    padding: 0;
  }

  dl dt {
    padding: 0;
    margin-top: 6px;
    font-size: 1em;
    font-style: italic;
    font-weight: bold;
  }

  dl dd {
    padding: 0 16px;
    margin-bottom: 16px;
  }

  blockquote {
    font-size: inherit;
    padding: 0 15px;
    color: #777;
    border-left: 4px solid #ddd;
  }

  blockquote>:first-child {
    margin-top: 20px;
  }

  blockquote>:last-child {
    margin-bottom: 20px;
  }

  blockquote {
    margin: 20px 0 !important;
    background-color: #f5f8fc;
    padding: 1rem;
    color: #8796A8;
    border-left: none;
  }

  table {
    display: block;
    width: 100%;
    overflow: auto;
    margin: 25px 0;
  }

  table th {
    font-weight: bold;
  }

  table th,
  table td {
    padding: 6px 13px;
    border: 1px solid #ddd;
  }

  table tr {
    background-color: #fff;
    border-top: 1px solid #ccc;
  }

  table tr:nth-child(2n) {
    background-color: #f8f8f8;
  }

  img {
    max-width: 100%;
    -moz-box-sizing: border-box;
    box-sizing: border-box;
  }

  img {
    border: 1px solid #ddd;
    box-shadow: 0 0 30px #ccc;
    -moz-box-shadow: 0 0 30px #ccc;
    -webkit-box-shadow: 0 0 30px #ccc;
    margin-bottom: 30px;
    margin-top: 10px;
  }

  code {
    background: rgba(90, 87, 87, 0);
    margin: 5px;
    color: #858080;
    border-radius: 4px;
    background-color: #f9fafa;
    border: 1px solid #e4e4e4;
    max-width: 740px;
    overflow-x: auto;
    font-size: .9em;
    padding: 1px 2px 1px;
  }

  code:before,
  code:after {
    letter-spacing: -0.2em;
    content: "\00a0";
  }

  pre>code {
    padding: 0;
    margin: 0;
    font-size: 100%;
    white-space: pre;
    background: transparent;
    border: 0;
  }

  .highlight {
    margin-bottom: 16px;
  }

  .highlight pre,
  pre {
    padding: 14px;
    overflow: auto;
    line-height: 1.45;
    // background-color: #4e4e4e;
    border-radius: 3px;
    color: inherit;
    border: none;
  }

  .highlight pre {
    margin-bottom: 0;
  }

  pre {
    word-wrap: normal;
  }

  pre code {
    display: block;
    padding: 0;
    margin: 0;
    overflow: initial;
    line-height: inherit;
    word-wrap: normal;
    background-color: transparent;
    border: 0;
  }

  pre code:before,
  pre code:after {
    content: normal;
  }
}
```

在主样式代码中对其进行加载，并书写页面结构的样式：

_resources/sass/app.scss_

```
.
.
.

/* Topic Show Page */

@import "topic_body";

.topics-show-page {

    .card {
        padding: 15px;

        h1 {
            margin: 0.4em auto 0.6em;
            font-size: 28px;
            line-height: 38px;
        }
    }
}
```

等待样式表编译完成后刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/H9ECDHiJEz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/H9ECDHiJEz.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "话题内容页面"
```



