## 站点首页

现在我们的首页还是一个接近空白的页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/sMCd6gX0xj.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/sMCd6gX0xj.png!large)

接下来我们将`/`路由直接使用话题列表来渲染。

## 修改路由

打开`routes/web.php`文件，修改文件里的第一行：

```
Route::get('/', 'PagesController@root')->name('root');
```

改为：

```
Route::get('/', 'TopicsController@index')->name('root');
```

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Jrvtv1TqqP.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/Jrvtv1TqqP.png!large)

样式有些问题，那是因为我们的话题列表页面是使用『路由专属样式类』区分的，新增路由样式类到原有的样式上即可。

寻找以下：

_resources/sass/app.scss_

```
.
.
.
/* Topic Index Page */
.topics-index-page, .categories-show-page {
.
.
.
}
.
.
.
```

修改为以下：

_resources/sass/app.scss_

```
.
.
.
/* Topic Index Page */
.topics-index-page, .categories-show-page, .root-page {
.
.
.
}
.
.
.
```

样式编译完成后，即可看到正常的话题列表：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/tCXJ7Pshwg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/26/1/tCXJ7Pshwg.png!large)

## Git 版本控制

下面把代码纳入到版本管理：

```
$ git add -A
$ git commit -m "首页话题列表"
```



