## 辅助函数

Laravel 提供了很多[辅助函数](https://learnku.com/docs/laravel/5.7/helpers)，有时候我们也需要创建自己的辅助函数。

我们把所有的『自定义辅助函数』存放于`app/helpers.php`文件中，这里需要新建一个空文件：

```
$ touch app/helpers.php
```

> Linux 的`touch`touch 命令有两个功能：一是用于把已存在文件的时间标签更新为系统当前的时间（默认方式），它们的数据将原封不动地保留下来；二是用来创建新的空文件。

在我们新增`helpers.php`文件之后，还需要在项目根目录下 composer.json 文件中的 autoload 选项里 files 字段加入该文件：

_composer.json_

```
{
    ...

    "autoload": {
        "psr-4": {
            "App\\": "app/"
        },
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "files": [
            "app/helpers.php"
        ]
    }
    ...
}
```

修改保存后运行以下命令进行重新加载文件即可：

```
$ composer dump-autoload
```

## Git 代码版本控制

接着让我们将该文件加入到版本控制中：

```
$ git add -A
$ git commit -m "新增自定义辅助函数文件"
```



