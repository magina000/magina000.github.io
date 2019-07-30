---
title: 使用gulp解决静态资源缓存问题
date: 2019-07-30 14:32:06
categories: 学习
tags: [工具]
---

### 实现办法：自动给 js、css 等静态资源后面加上版本号（gulp 版本 3.9.1）

_使用前_![](prev.png)_使用后_![](next.png)

默认已经安装 Node.js 和 npm

##### **全局安装 gulp**

```
cnpm install gulp -g
```

##### **在当前项目中安装 gulp**

```
cnpm install --save-dev gulp
```

查看 gulp 是否安装成功

```
gulp -v
```

出现版本号表示安装成功 CLI 表示全局版本；Local 本地版本（建议一致）

##### **安装三个 gulp 插件 gulp**

```
npm install --save-dev gulp-rev
npm install --save-dev gulp-rev-collector
npm install --save-dev run-sequence
```

#### **下面的文件修改重要！！！**

###### > node_modules/gulp-rev/index.js

修改第 134 行

```
// manifest[originalFile] = revisionedFile;
manifest[originalFile] = originalFile + '?v=' + file.revHash;
```

###### > node_modules/gulp-rev/node_modules/rev-path/index.js

修改第 9 行

```
// return modifyFilename(pth, (filename, ext) =&gt; `${filename}-${hash}${ext}`);
return modifyFilename(pth, (filename, ext) =&gt; `${filename}${ext}`);
```

###### > node_modules/gulp-rev-collector/index.js

修改第 40 行

```
// var cleanReplacement =  path.basename(json[key]).replace(new RegExp( opts.revSuffix ), '' );
var cleanReplacement =  path.basename(json[key]).split('?')[0];
```

修改第 139 行

```
// regexp: new RegExp(  dirRule.dirRX + pattern, 'g' ),
regexp: new RegExp(  dirRule.dirRX + pattern +'(\\?v=\\w{10})?', 'g' ),
```

第 164 行

```
// regexp: new RegExp( prefixDelim + pattern, 'g' ),
regexp: new RegExp( prefixDelim + pattern+'(\\?v=\\w{10,})?', 'g' ),
```

##### **新建一个 gulpfile.js 文件**

_内容如下_

```
//引入gulp和gulp插件
var gulp = require('gulp'),
    runSequence = require('run-sequence'),
    rev = require('gulp-rev'),
    revCollector = require('gulp-rev-collector');

//定义css、js源文件路径。这里填写自己真实的路径
var cssSrc = '[你的真实项目地址]/static/css/*.css',
    jsSrc = '[你的真实项目地址]/static/js/*.js';

//CSS生成文件hash编码并生成 rev-manifest.json文件名对照映射
gulp.task('revCss', function(){
    return gulp.src(cssSrc)
        .pipe(rev())
        .pipe(rev.manifest())
        //.pipe(minifycss())   //压缩css，需要新的插件，下载速度太慢，我放弃了
        .pipe(gulp.dest('rev/css'));
});

//js生成文件hash编码并生成 rev-manifest.json文件名对照映射
gulp.task('revJs', function(){
    return gulp.src(jsSrc)
        .pipe(rev())
        .pipe(rev.manifest())
        //.pipe(uglify())    ////压缩JS，需要新的插件，下载速度太慢，我放弃了
        .pipe(gulp.dest('rev/js'));
});

//Html替换css、js文件版本
gulp.task('revHtml', function () {
    return gulp.src(['rev/**/*.json', '[你的真实项目地址]/**/view/**/*.html'])//填写自己的真实模板存放位置
        .pipe(revCollector())
        .pipe(gulp.dest('[你的真实项目地址]/application'));
});

//task合并顺序执行
gulp.task('dev', function (done) {
    condition = false;
    runSequence(
        ['revCss'],
        ['revJs'],
        ['revHtml'],
        done);
});

gulp.task('default', ['dev']);
```

##### **执行**

_在项目根目录输入 gulp 就 OK 了_
