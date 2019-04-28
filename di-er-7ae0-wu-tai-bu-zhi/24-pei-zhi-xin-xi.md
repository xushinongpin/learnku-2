## 存放位置

Laravel 框架的所有配置都保存在`config`目录中，每个文件中包含多个配置选项，每个选项都有说明，你可随时查看这些文件并熟悉都有哪些配置选项可供你使用。

## 配置文件

下面是配置文件的简单说明：

| 文件名称 | 配置类型 |
| :--- | :--- |
| app.php | 应用相关，如项目名称、时区、语言等 |
| auth.php | 用户授权，如用户登录、密码重置等 |
| broadcasting.php | 事件广播系统相关配置 |
| cache.php | 缓存相关配置 |
| database.php | 数据库相关配置，包括 MySQL、Redis 等 |
| filesystems.php | 文件存储相关配置 |
| hashing.php | 加密算法相关设置 |
| logging.php | 日志记录相关的配置 |
| mail.php | 邮箱发送相关的配置 |
| queue.php | 队列系统相关配置 |
| services.php | 放置第三方服务配置 |
| session.php | 用户会话相关配置 |
| view.php | 视图存储路径相关配置 |

## 访问配置值

你可以在应用程序的任何位置使用全局`config`函数来访问配置值。配置值的访问可以使用「点」语法，这其中包含了要访问的_文件名称_和_选项_的名称。还可以指定默认值，如果配置选项不存在，则返回默认值：

```
$value = config('app.timezone');
```

要在运行时设置配置值，传递一个数组给 config 函数：

```
config(['app.timezone' => 'America/Chicago']);
```

## 调整配置信息

现在我们需要调整几个应用配置信息。

### 应用名称

_config/app.php_

```
'name' => env('APP_NAME', 'Laravel'),
```

此处使用`env()`方法可知其优先读取`.env`文件中的`APP_NAME`选项，前往修改为`LaraBBS`：

_.env_

```
APP_NAME=LaraBBS

.
.
.
```

### 应用链接

_config/app.php_

```
'url' => env('APP_URL', 'http://localhost'),
```

优先读取`.env`文件中的`APP_URL`选项，前往修改为以下：

_.env_

```
.
.
.
APP_URL=http://larabbs.test
.
.
.
```

### 时区

请将选项`timezone`修改为上海的时区：

_config/app.php_

```
'timezone' => 'Asia/Shanghai',
```

### 默认语言

请将选项`locale`修改为中文：

_config/app.php_

```
'locale' => 'zh-CN',
```

## Git 代码版本控制

接着让我们将该文件加入到版本控制中：

```
$ git add -A
$ git commit -m "修改配置信息"
```



