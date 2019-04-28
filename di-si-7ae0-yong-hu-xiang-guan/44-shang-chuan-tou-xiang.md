## 上传头像

目前为止，图中的这两张图都是测试图片，接下来我们将一起开发个人资料里的头像上传功能，并将这两张图片换为用户上传的头像。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/EFf5mpnx3K.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/EFf5mpnx3K.png!large)

## 模型文件修改

首先我们需在`User`模型里将`avatar`字段加入到允许修改的白名单`$fillable`中：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/jb5R69AfDv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/jb5R69AfDv.png!large)

## 编辑页面

接下来我们在[资料编辑页面](http://larabbs.test/users/1/edit)的『个人简介』编辑框下面，增加头像上传的选项：

_resources/views/users/edit.blade.php_

```
.
.
.
          <div class="form-group">
            <label for="introduction-field">个人简介</label>
            <textarea name="introduction" id="introduction-field" class="form-control" rows="3">{{ old('introduction', $user->introduction) }}</textarea>
          </div>

          <div class="form-group mb-4">
            <label for="" class="avatar-label">用户头像</label>
            <input type="file" name="avatar" class="form-control-file">

            @if($user->avatar)
              <br>
              <img class="thumbnail img-responsive" src="{{ $user->avatar }}" width="200" />
            @endif
          </div>
.
.
.
```

效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9TNbnIucnm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9TNbnIucnm.png!large)

在 Laravel 中，我们可直接通过[请求对象（Request）](https://learnku.com/docs/laravel/5.7/requests#retrieving-uploaded-files)来获取用户上传的文件，如以下两种方法：

```
// 第一种方法
$file = $request->file('avatar');

// 第二种方法，可读性更高
$file = $request->avatar;
```

接下来我们将在 UsersController 的`update()`方法中，利用 Laravel 的`dd()`调试方法，来查看文件上传的情况：

_app/Http/Controllers/UsersController.php_

```
<?php
.
.
.
class UsersController extends Controller
{
    .
    .
    .
    public function update(UserRequest $request, User $user)
    {
        dd($request->avatar);

        $user->update($request->all());
        return redirect()->route('users.show', $user->id)->with('success', '个人资料更新成功！');
    }
}
```

测试一下：

1. 访问[资料编辑页面](http://larabbs.test/users/1/edit)；
2. 点击 『choose file』 按钮，选择图片；
3. 点击『保存』按钮提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/7iJROJJiVy.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/7iJROJJiVy.png!large)

打印出来是图片文件名称的字符串，与我们的预期不符，图片上传后通过`$request`取到的应是图片对象。

经过一番仔细检查后，发现是因为我们忘记为表单添加`enctype="multipart/form-data"`声明了。请记住，在图片或者文件上传时，为表单添加此句声明是必须的。那我们再次修改下：

_resources/views/users/edit.blade.php_

```
.
.
.
        <form action="{{ route('users.update', $user->id) }}" method="POST" 
          accept-charset="UTF-8" 
          enctype="multipart/form-data">
.
.
.
```

重新测试一下：

1. 访问
   [资料编辑页面](http://larabbs.test/users/1/edit)
   ；
2. 点击 『choose file』 按钮，选择图片；
3. 点击『保存』按钮提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Q1KIMF9nZh.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Q1KIMF9nZh.png!large)

这次对了，可以看到是一个对象打印出来。Laravel 的『用户上传文件对象』底层使用了 Symfony 框架的[UploadedFile](http://api.symfony.com/3.0/Symfony/Component/HttpFoundation/File/UploadedFile.html)对象进行渲染，为我们提供了便捷的文件读取和管理接口，我们将在后面使用这些方法。

现在我们已经能得到用户上传的图片数据了，接下来是对图片进行存储。

## 存储用户上传图片

本项目中，我们不止上传头像需要用到『图片上传功能』，在后面发布帖子功能中，我们也将会允许用户上传图片，所以此处我们需要预先设计一下图片上传相关的逻辑，我们可以将『图片上传』核心操作做成一个工具类：

_app/Handlers/ImageUploadHandler.php_

    <?php

    namespace App\Handlers;

    class ImageUploadHandler
    {
        // 只允许以下后缀名的图片文件上传
        protected $allowed_ext = ["png", "jpg", "gif", 'jpeg'];

        public function save($file, $folder, $file_prefix)
        {
            // 构建存储的文件夹规则，值如：uploads/images/avatars/201709/21/
            // 文件夹切割能让查找效率更高。
            $folder_name = "uploads/images/$folder/" . date("Ym/d", time());

            // 文件具体存储的物理路径，`public_path()` 获取的是 `public` 文件夹的物理路径。
            // 值如：/home/vagrant/Code/larabbs/public/uploads/images/avatars/201709/21/
            $upload_path = public_path() . '/' . $folder_name;

            // 获取文件的后缀名，因图片从剪贴板里黏贴时后缀名为空，所以此处确保后缀一直存在
            $extension = strtolower($file->getClientOriginalExtension()) ?: 'png';

            // 拼接文件名，加前缀是为了增加辨析度，前缀可以是相关数据模型的 ID 
            // 值如：1_1493521050_7BVc9v9ujP.png
            $filename = $file_prefix . '_' . time() . '_' . str_random(10) . '.' . $extension;

            // 如果上传的不是图片将终止操作
            if ( ! in_array($extension, $this->allowed_ext)) {
                return false;
            }

            // 将图片移动到我们的目标存储路径中
            $file->move($upload_path, $filename);

            return [
                'path' => config('app.url') . "/$folder_name/$filename"
            ];
        }
    }

> 注：请仔细阅读代码注释

我们将使用`app/Handlers`文件夹来存放本项目的工具类，『工具类（utility class）』是指一些跟业务逻辑相关性不强的类，`Handlers`意为**处理器**，ImageUploadHandler 意为图片上传处理器，简单易懂。

接下来我们需要在 UsersController 里调用（注意顶部`use`引入）：

_app/Http/Controllers/UsersController.php_

```
<?php
.
.
.
use App\Handlers\ImageUploadHandler;

class UsersController extends Controller
{
    .
    .
    .

    public function update(UserRequest $request, ImageUploadHandler $uploader, User $user)
    {
        $data = $request->all();

        if ($request->avatar) {
            $result = $uploader->save($request->avatar, 'avatars', $user->id);
            if ($result) {
                $data['avatar'] = $result['path'];
            }
        }

        $user->update($data);
        return redirect()->route('users.show', $user->id)->with('success', '个人资料更新成功！');
    }
}
```

* 因为我们使用了命名空间，所以需要在顶部加载 use App\Handlers\ImageUploadHandler;；
* $data = $request-&gt;all\(\); 赋值 $data 变量，以便对更新数据的操作；
* 以下代码处理了图片上传的逻辑，注意 if \($result\) 的判断是因为 ImageUploadHandler 对文件后缀名做了限定，不允许的情况下将返回 false：

```
if ($request->avatar) {
    $result = $uploader->save($request->avatar, 'avatars', $user->id);
    if ($result) {
        $data['avatar'] = $result['path'];
    }
}
```

* $user-&gt;update\($data\); 这一步才是执行更新。

接下来让我们来测试一下：

1. 访问[资料编辑页面](http://larabbs.test/users/1/edit)；
2. 点击 『choose file』 按钮，选择图片；
3. 点击『保存』按钮提交表单。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/5BRqQuYB0W.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/5BRqQuYB0W.png!large)

提交成功，打开项目文件夹，一步步寻找下去，找到我们刚刚上传的图片：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/hni56Fr2bB.jpg!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/hni56Fr2bB.jpg!large)

图片上传成功了，接下来我们做数据嵌套，将我们的头像显示出来。

## Git 版本控制

我们在上传图片的时候，程序自动创建了`public/uploads/images/avatars/`目录，此文件夹下的文件皆为用户上传的头像文件，我们需要防止这些文件被纳入 Git 版本控制器中，可以利用 Git 的`.gitignore`机制来实现：

_public/uploads/images/avatars/.gitignore_

```
*
!.gitignore
```

上面的两行代码意为：当前文件夹下，忽略所有文件，除了`.gitignore`。

下面把代码纳入到版本管理，为下一节做好准备：

```
$ git add -A
$ git commit -m "上传头像"
```



