---
layout: post
title: "通过XHR上传浏览器端的文件资源"
comments: true
tags: [Tech, Javascript, Chrome-Extension]
description: "通过发起XHR请求的方式获取浏览器端的文件资源"
---

最近在为公司开发一个[海淘的Chrome扩展][haitaobao-link]，扩展的需求之一是：获取当前页面中的某张图片，并将其上传至公司的服务器保存。

当时的第一思路是：

 - 获取图片的URL，然后把URL提交给服务器，让服务器进行下载操作
 - 如果服务器访问图片资源时被拒绝，就需要让用户手动下载该图片，然后手动上传至我们的服务器

不过在后来的实践中发现，其实可以直接通过**发起[XHR][xhr-link]请求并指定其返回类型**的方式来获取浏览器返回的raw数据(`Blob`)，然后将这些raw数据当作文件上传至服务器。

<!--more-->

*在这里首先要说明一点是的：这种将XHR的响应内容当作文件上传的思路，通常是需要[跨站请求资源][cross-site-access](Cross-site HTTP requests)的，所以除非是在允许跨站请求的运行环境中（如Chrome扩展内）运行这些JS，否则这种思路几乎不可用*

###思路
这种上传客户端资源的解决方案最先来自这里[upload-a-file-in-a-google-chrome-extension][sf-link]。基本思路是：当使用`XHR`向某个URL发起请求时（如请求某个`image`、`css`、`js`文件资源）我们可以设定它的返回类型（`responseType`）；当我们把返回类型设置为`blob`时，我们便可以直接把返回的raw数据当作"文件"来上传到服务器，以此来实现客户端访问资源的上传。

----------

###怎么做
1) 通过XHR请求一个图片资源
    
    var imgURL = 'http://lyfing.qiniudn.com/external_links/pocker_cards_family.png';
    var xhr = new XMLHttpRequest(); 
    xhr.open("GET", imgURL, true);
    xhr.responseType = "blob";
    xhr.onload = function(){
        var blob = xhr.response;
        var msg = ' Request URL = \n ' + imgURL;
        msg += '\n\n Get Response !';
        msg += '\n responseType = ' + blob.type;
        msg += '\n responseSize = ' + Math.round(blob.size / 1024) + 'KB';
        alert(msg);
    };
    xhr.send();

<script type="text/javascript">
function getIMGRes(){
    var imgURL = 'http://lyfing.qiniudn.com/external_links/pocker_cards_family.png';
    var xhr = new XMLHttpRequest(); 
    xhr.open("GET", imgURL, true);
    xhr.responseType = "blob";
    xhr.onload = function(){
        var blob = xhr.response;
        var msg = ' Request URL = \n ' + imgURL;
        msg += '\n\n Get Response !';
        msg += '\n responseType = ' + blob.type;
        msg += '\n responseSize = ' + Math.round(blob.size / 1024) + 'KB';
        alert(msg);
    };
    xhr.send();
};
</script>

<input type="button" onclick="getIMGRes();" value="点击测试">

2) 将返回的[Blob][what-is-blob]对象上传至服务器
    
    // 直接使用上一步获得的blob对象
    var blob = xhr1.response;
    // 我们可以直接使用FormData构建Form表单
    var formData = new FormData();
    // 获取文件类型(例如: blob.type = 'image/png')
    var fileType = blob.type.split('/')[1]; 
    // 将返回的blob对象直接当作文件上传，并设置上传文件名为 test001.fileType
    formData.append('file', blob, 'test001.' + fileType);
    
    var xhr2 = new XMLHttpRequest();
    
    xhr2.upload.onprogress = function(event){
        ...
        $msg.html('正在上传 ( ' + percent + '% )');
    }
    
    xhr2.onload = function(){
        if ( !confirm('上传完成！即将跳转至上传结果页...') ) return;
        // 返回结果是一段纯文本，我们用正则取出本次上传的信息链接
        var url = xhr2.response.match(/http:\/\/[^\s]+/)[0];
        window.open(url);
    }
    
    xhr2.open('POST', 'http://posttestserver.com/post.php?dir=example', true);
    xhr2.send(formData);

<script type="text/javascript">
function upload(){
    var imgURL = 'http://lyfing.qiniudn.com/external_links/pocker_cards_family.png';
    var xhr1 = new XMLHttpRequest();
    xhr1.open("GET", imgURL, true);
    xhr1.responseType = "blob";
    xhr1.onload = function(){
        var blob = xhr1.response;
        var formData = new FormData();
        var fileType = blob.type.split('/')[1];
        formData.append('file', blob, 'test001.' + fileType);

        var xhr = new XMLHttpRequest();
        
        xhr.upload.onprogress = function(event){
            var $msg2 = $('#part2Msg');
            var percent = Math.floor(event.position / event.totalSize * 100);
            $msg2.html('正在上传 ( ' + percent + '% )');
        }

        xhr.onload = function(){
            if ( !confirm('上传完成！即将跳转至上传结果页...') ) return;
            var url = xhr.response.match(/http:\/\/[^\s]+/)[0];
            window.open(url);
        }
        
        xhr.open('POST', 'http://posttestserver.com/post.php?dir=example', true);
        xhr.send(formData);        
    };
    xhr1.send(); 
}
</script>

<input type="button" onclick="upload();" value="点击测试">&nbsp;<span id="part2Msg"></span>

----------

### 用到的一些`对象`
上面我们用到了`FormData`对象([详情][form-data-link])，它是一个模拟了HTML中的Form表单的实体，你可以直接使用它来构建`key/value pair`，其中的`value`可以是`文件`、`Blob`或者纯`String`。下面做一个简单的使用示例（Chrome、Firefox可随便使用，IE10+才可使用)

    // 创建Form表单对象formData，可在构造函数中传入一个HTML表单对象htmlForm
    // 它的键值对会被添加到formData中
    var formData = new FormData();
    // 添加一个键值对
    formData.append('userID', '112233');
    // 添加一个文件对象
    var file = document.getElementByID('fileField');
    formData.append('file1', file);
    // 添加一个Blob对象，它将被视为文件上传
    formData.append('file2', blob);

关于`Blob`对象，它是一个类似于`文件`的结构体，事实上`文件`接口正是在它的基础上做的扩展。[更多关于Blob][blob-link]

--------
**备注**：

 - [跨站请求HTTP资源][xhr-link]
 
 - [`Blob`][blob-link]对象
 
 - [`FormData`][form-data-link]对象

 - 示例中用到的文件上传服务：http://posttestserver.com/post.php?dir=example

 - 示例中用到的图片：![](http://lyfing.qiniudn.com/external_links/pocker_cards_family.png)


[haitaobao-link]:https://chrome.google.com/webstore/detail/obnbgneldjmmpgkbnnbiiinijmiclpaa
[xhr-link]:https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest
[sf-link]:http://stackoverflow.com/a/10002486/1241980
[cross-site-access]:https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS
[cache-control]:http://stackoverflow.com/a/4480318/1241980
[what-is-blob]:https://developer.mozilla.org/en-US/docs/Web/API/Blob
[form-data-link]:https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/FormData
[blob-link]:https://developer.mozilla.org/en-US/docs/Web/API/Blob
