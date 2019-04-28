## 上传图片

接下来我们要为话题发布页面添加图片上传功能。

## 1. 设置路由

我们需要先设置路由，这样接下来我们才能设置上传图片的 URL：

_routes/web.php_

```
.
.
.

Route::post('upload_image', 'TopicsController@uploadImage')->name('topics.upload_image');
```

## 2. JS 脚本调用

我们已经在前面的章节中把`uploader.js`脚本加载进来了，接下来只需要按照[编辑器上传图片文档](http://simditor.tower.im/docs/doc-config.html#anchor-upload)进行设定即可：

_resources/views/topics/create\_and\_edit.blade.php_

```
.
.
.
@section('scripts')
  <script type="text/javascript" src="{{ asset('js/module.js') }}"></script>
  <script type="text/javascript" src="{{ asset('js/hotkeys.js') }}"></script>
  <script type="text/javascript" src="{{ asset('js/uploader.js') }}"></script>
  <script type="text/javascript" src="{{ asset('js/simditor.js') }}"></script>

  <script>
    $(document).ready(function() {
      var editor = new Simditor({
        textarea: $('#editor'),
        upload: {
          url: '{{ route('topics.upload_image') }}',
          params: {
            _token: '{{ csrf_token() }}'
          },
          fileKey: 'upload_file',
          connectionCount: 3,
          leaveConfirm: '文件上传中，关闭此页面将取消上传。'
        },
        pasteImage: true,
      });
    });
  </script>
@stop

```

参数解释：

* `pasteImage`
  —— 设定是否支持图片黏贴上传，这里我们使用 true 进行开启；
* `url`
  —— 处理上传图片的 URL；
* `params`
  —— 表单提交的参数，Laravel 的 POST 请求必须带防止 CSRF 跨站请求伪造的
  `_token`
  参数；
* `fileKey`
  —— 是服务器端获取图片的键值，我们设置为
  `upload_file`
  ;
* `connectionCount`
  —— 最多只能同时上传 3 张图片；
* `leaveConfirm`
  —— 上传过程中，用户关闭页面时的提醒。

## 3. 控制器图片处理

页面调用已经准备好，接下来是编写服务器端的逻辑。根据[编辑器上传图片文档](http://simditor.tower.im/docs/doc-config.html#anchor-upload)，服务器端通过返回以下 JSON 来反馈上传状态：

```
{
  "success": true/false,
  "msg": "error message", # optional
  "file_path": "[real file path]"
}
```

在 Laravel 的控制器方法中，如果直接返回数组，将会被自动解析为 JSON。接下来我们编写控制器的逻辑，根据第一步设置的路由在话题控制器中新增`uploadImage`方法：

_app/Http/Controllers/TopicsController.php_

```
<?php
.
.
.

use App\Handlers\ImageUploadHandler;

class TopicsController extends Controller
{
    .
    .
    .
    public function uploadImage(Request $request, ImageUploadHandler $uploader)
    {
        // 初始化返回数据，默认是失败的
        $data = [
            'success'   => false,
            'msg'       => '上传失败!',
            'file_path' => ''
        ];
        // 判断是否有上传文件，并赋值给 $file
        if ($file = $request->upload_file) {
            // 保存图片到本地
            $result = $uploader->save($request->upload_file, 'topics', \Auth::id(), 1024);
            // 图片保存成功的话
            if ($result) {
                $data['file_path'] = $result['path'];
                $data['msg']       = "上传成功!";
                $data['success']   = true;
            }
        }
        return $data;
    }
}
```

> 请注意在文件顶部引入`ImageUploadHandler`。

图片上传使用的是与头像上传同一个接口，因为我们提前考虑了代码重用，这里的开发异常轻松。

权限控制上，我们只允许登录用户才能上传图片。控制器的`__construct`里已做了中间件认证，默认需要登录权限，因此我们不需要做任何修改。

## 4. 测试一下

接下来可以尝试一下图片上传：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/oxJoj51u4x.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/oxJoj51u4x.gif!large)

测试下从剪贴板里黏贴图片：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/6XLt7s9UGf.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/24/1/6XLt7s9UGf.gif!large)

拖拽图片上传也是支持的，请自行测试。下一章节我们开发话题详情页，将我们上传的图片显示出来。

## Git 版本控制

同头像上传类似的，我们在上传图片的时候，程序自动创建了`public/uploads/images/topics/`目录，此文件夹下的文件皆为用户上传的话题图片文件，我们需要防止这些文件被纳入 Git 版本控制器中，可以利用 Git 的`.gitignore`机制来实现：

_public/uploads/images/topics/.gitignore_

```
*
!.gitignore
```

上面的两行代码意为：当前文件夹下，忽略所有文件，除了`.gitignore`。

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "用户可以上传话题图片"
```



