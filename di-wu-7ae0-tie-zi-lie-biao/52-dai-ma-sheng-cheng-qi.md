## 说明

在『Laravel 教程』系列课程中，我们开发时遵守的代码风格是[Laravel 项目开发规范](https://learnku.com/courses/laravel-specification)。遵照此规范，在实际操作中，有许多重复，接下来推荐一款专为此规范量身定制的代码生成器 ——[Laravel 5.x Scaffold Generator](https://github.com/summerblue/generator)。代码生成器能让你通过执行一条 Artisan 命令，完成注册路由、新建模型、新建表单验证类、新建资源控制器以及所需视图文件等任务，不仅约束了项目开发的风格，还能极大地提高我们的开发效率。

在接下来的 LaraBBS 项目开发中，我们将利用此扩展来快速构建项目原型。

> 后续章节我们将『Laravel 5.x Scaffold Generator』简称为『代码生成器』，或者『生成器』。

## 安装

### 1. 通过 Composer 安装

```
$ composer require "summerblue/generator:~1.0" --dev
```

后置参数`--dev`表明我们只在开发环境中使用。

### 2. 版本标记

本小节剩下的篇幅中，我们将带大家来测试下代码生成器，熟悉基本操作。在测试前，我们先把代码纳入到版本管理，做个版本标记，以便后续测试的代码回滚：

```
$ git add -A
$ git commit -m "新增 generator 扩展"
```

## 3. 试运行

接下来我们可以放心的测试，运行[generator](https://github.com/summerblue/generator)的 readme 里提供的示例：

```
$ php artisan make:scaffold Projects --schema="name:string:index,description:text:nullable,subscriber_count:integer:unsigned:default(0)"
```

我们在命令行中指定了数据模型还有具体的字段信息，运行命令后，generator 帮我们生成了一堆文件，并在最后执行了数据库迁移：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/7HRTmvwdNT.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/7HRTmvwdNT.png!large)

我们将会在下一个章节中具体讲解所有生成的文件，接下来让我们还原项目到正常状态，首先回滚数据库迁移：

```
$ php artisan migrate:rollback
```

还原修改文件到原始状态：

```
$ git checkout . 
```

查看文件修改状态：

```
$ git status
```

输出：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lyhkFy7xUb.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lyhkFy7xUb.png!large)

可以看出剩下的是新建的文件，接下来使用下面命令还原：

```
$ git clean -f -d 
```

命令`git clean`作用是清理项目，`-f`是强制清理文件的设置，`-d`选项命令连文件夹一并清除。执行成功后再使用：

```
$ git status
```

即可看到`nothing to commit, working directory clean`的提示信息，表示项目文件已经清理干净，我们可以继续下一章节的学习。

