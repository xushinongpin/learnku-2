## 裁剪图片

我们还有一个地方要优化，用户有时会上传分辨率较大的图片，类似以下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/kwRIEfLsaK.jpeg!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/kwRIEfLsaK.jpeg!large)

而我们个人空间里显示区域最大也就 208px，即使要兼容[视网膜屏幕（Retina Screen）](https://baike.baidu.com/item/%E8%A7%86%E7%BD%91%E8%86%9C%E5%B1%8F%E5%B9%95)的话，最多也就需要 208px \* 2 = 416px 。图片太大会拖慢页面的加载速度，所以接下来我们将对此进行优化。

我们将使用备受欢迎的[Intervention/image](https://github.com/Intervention/image)扩展包来处理图片裁切的逻辑，接下来我们需要先安装此扩展包；

## 1. 安装扩展包

1. Composer 安装

```
$ composer require intervention/image
```

1. 配置信息

执行以下命令获取配置信息：

```
$ php artisan vendor:publish --provider="Intervention\Image\ImageServiceProviderLaravel5"
```

结果如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cX4F0X6apE.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/cX4F0X6apE.png!large)

打开`config/image.php`文件可以看到只有一个驱动器的选项，支持的值有[GD 库](http://php.net/manual/zh/intro.image.php)和[ImageMagic](http://php.net/manual/zh/intro.imagick.php)：

[![](https://iocaffcdn.phphub.org/uploads/images/201709/22/1/Q2lKDNZIPq.png "file")](https://iocaffcdn.phphub.org/uploads/images/201709/22/1/Q2lKDNZIPq.png)

> 注：此处我们使用默认的`gd`即可，如果将要开发的项目需要较专业的图片，请考虑 ImageMagic。

## 2. 开始裁剪

我们将裁切的逻辑写在`ImageUploadHandler`中，请将以下代码替换：

_app/Handlers/ImageUploadHandler.php_

    <?php

    namespace App\Handlers;

    use Image;

    class ImageUploadHandler
    {
        protected $allowed_ext = ["png", "jpg", "gif", 'jpeg'];

        public function save($file, $folder, $file_prefix, $max_width = false)
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

            // 如果限制了图片宽度，就进行裁剪
            if ($max_width && $extension != 'gif') {

                // 此类中封装的函数，用于裁剪图片
                $this->reduceSize($upload_path . '/' . $filename, $max_width);
            }

            return [
                'path' => config('app.url') . "/$folder_name/$filename"
            ];
        }

        public function reduceSize($file_path, $max_width)
        {
            // 先实例化，传参是文件的磁盘物理路径
            $image = Image::make($file_path);

            // 进行大小调整的操作
            $image->resize($max_width, null, function ($constraint) {

                // 设定宽度是 $max_width，高度等比例双方缩放
                $constraint->aspectRatio();

                // 防止裁图时图片尺寸变大
                $constraint->upsize();
            });

            // 对图片修改后进行保存
            $image->save();
        }
    }

> 注：请仔细阅读代码注释，此次新增`reduceSize()`方法，以及此方法的调用。

以上的`save()`方法中，我们新增了`$max_width`参数，用来指定最大图片宽度，我们修改 UsersController 的`update()`方法中的调用，修改为：

```
$result = $uploader->save($request->avatar, 'avatars', $user->id, 416);
```

如下图：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Wga3n1eHmx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/Wga3n1eHmx.png!large)

## 开始测试

进入[资料编辑页面](http://larabbs.test/users/1/edit)，选择一张较大的图片（[示例图片下载](https://iocaffcdn.phphub.org/uploads/images/201709/22/1/zzgDzs4TjD.png)），然后点击保存提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lCRIuG26U7.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/lCRIuG26U7.png!large)

可以看到更新成功的提示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/2Kf7Aoh6ts.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/2Kf7Aoh6ts.png!large)

打开文件夹查看我们刚刚上传的文件：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9xVjtp5Z7L.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/1/9xVjtp5Z7L.png!large)

至此图片上传功能开发完毕。

## Git 代码版本控制

接着让我们将本次更改纳入版本控制中：

```
$ git add -A
$ git commit -m "裁剪头像"
```



