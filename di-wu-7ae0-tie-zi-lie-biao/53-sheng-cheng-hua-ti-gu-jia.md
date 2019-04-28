## 说明

天下武功，唯快不破，Laravel 最吸引人的地方，就是其开发上的高效。这一节我们来利用起上一章节安装的代码生成器，快速构建我们的帖子原型。

## 整理字段

在使用代码生成器之前，我们需要先整理好`topics`表的字段名称和字段类型：

| 字段名称 | 描述 | 字段类型 | 加索引缘由 | 其他 |
| :--- | :--- | :--- | :--- | :--- |
| `title` | 帖子标题 | 字符串（String） | 文章搜索 | 无 |
| `body` | 帖子内容 | 文本（text） | 不需要 | 无 |
| `user_id` | 用户 ID | 整数（int） | 数据关联 | `unsigned()` |
| `category_id` | 分类 ID | 整数（int） | 数据关联 | `unsigned()` |
| `reply_count` | 回复数量 | 整数（int） | 文章回复数量排序 | `unsigned()`,`default(0)` |
| `view_count` | 查看总数 | 整数（int） | 文章查看数量排序 | `unsigned()`,`default(0)` |
| `last_reply_user_id` | 最后回复的用户 ID | 整数（int） | 数据关联 | `unsigned()`,`default(0)` |
| `order` | 可用来做排序使用 | 整数（int） | 不需要 | `default(0)` |
| `excerpt` | 文章摘要，SEO 优化时使用 | 文本（text） | 不需要 | `nullable()` |
| `slug` | SEO 友好的 URI | 字符串（String） | 不需要 | `nullable()` |

* `unsigned()`
  —— 设置整数类型为非负整数 \(或称之为无符号整数\)
* `default()`
  —— 为字段添加默认值。
* `nullable()`
  —— 可为空

以上字段，拼接为最终命令：

```
$ php artisan make:scaffold Topic --schema="title:string:index,body:text,user_id:integer:unsigned:index,category_id:integer:unsigned:index,reply_count:integer:unsigned:default(0),view_count:integer:unsigned:default(0),last_reply_user_id:integer:unsigned:default(0),order:integer:unsigned:default(0),excerpt:text:nullable,slug:string:nullable"
```

执行结果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/S7qlT6AjQ9.jpeg!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/S7qlT6AjQ9.jpeg!large)

命令`make:scaffold`的`--schema`选项中是以逗号分隔的字段名称和设置信息，字段分别如下，对于上面的字段表格：

```
title:string:index
body:text
user_id:integer:unsigned:index
category_id:integer:unsigned:index
reply_count:integer:unsigned:default(0)
view_count:integer:unsigned:default(0)
last_reply_user_id:integer:unsigned:default(0)
order:integer:unsigned:default(0)
excerpt:text:nullable
slug:string:nullable
```

接下来我们看下代码生成器为我们做了哪些事情：

1. 创建话题的数据库迁移文件 —— 2018\_12\_23\_104258\_create\_topics\_table.php；
2. 创建话题数据工厂文件 —— TopicFactory.php；
3. 创建话题数据填充文件 —— TopicsTableSeeder.php；
4. 创建模型基类文件 —— Model.php， 并创建话题数据模型；
5. 创建话题控制器 —— TopicsController.php；
6. 创建表单请求的基类文件 —— Request.php，并创建话题表单请求验证类；
7. 创建话题模型事件监控器
   `TopicObserver`
   并在
   `AppServiceProvider`
   中注册；
8. 创建授权策略基类文件 —— Policy.php，同时创建话题授权类，并在
   `AuthServiceProvider`
   中注册；
9. 在 web.php 中更新路由，新增话题相关的资源路由；
10. 新建符合资源控制器要求的三个话题视图文件，并存放于
    `resources/views/topics`
    目录中；
11. 执行了数据库迁移命令
    `artisan migrate`
    ；
12. 因此次操作新建了多个文件，最终执行
    `composer dump-autoload`
    来生成 classmap。

目前为止，除了第 7 的`监控器`部分功能我们还未讲解以外，其他功能我们都已在前面章节和[Laravel 实战入门](https://learnku.com/courses/laravel-essential-training/5.5)中学习过。我们将在后面开发涉及到时，再讲解各个新建文件的功能。可以确认的是，在本章结束时，你将清楚地知道以上所有新建文件的功能，并从中体验到代码生成器带来的便利。

## 开始测试

此时打开[http://larabbs.test/topics](http://larabbs.test/topics)，查看话题列表页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Gk5NWrL84z.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Gk5NWrL84z.png!large)

打开[http://larabbs.test/topics/create](http://larabbs.test/topics/create)，未登录用户会被要求登录，登录成功后会跳转到：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/IQYHmpkjRm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/IQYHmpkjRm.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "生成话题骨架"
```



