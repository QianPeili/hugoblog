---
title: "Nginx Upload Module的编译和使用"
date: 2018-07-18T15:20:36+08:00
categories: 
 - note
tags: 
 - nginx
---

服务端需要增加一个文件上传功能，最直接可能就是后端程序里加个接收文件并保存的功能。但是之前看到 nginx 有个上传模块，而且如果直接 nginx 层把文件保存下来，就不需要再把文件转发到后端层处理了。

那么就开始吧。

### 一、下载 Nginx Upload Module 源码

模块的 github 地址： [fdintino/nginx-upload-module: A module for nginx web server for handling file uploads using multipart/form-data encoding (RFC 1867).](https://github.com/fdintino/nginx-upload-module/tree/master)

	git clone https://github.com/fdintino/nginx-upload-module.git

### 二、重新编译并重启 Nginx

为了将新模块添加到 nginx 程序中，需要重新编译 nginx。

`nginx -V` 可以输出原先的 nginx 所用的编译参数，原样复制后，添加新参数 `--add-module=/path/to/nginx-upload-module`。如：

	./configure <old configure options> --add-module=../nginx-upload-module

`make & make install` 之后，新的 nginx 二进制文件会替换原有的二进制文件，而原有的二进制文件会被重命名为 `nginx.old`。（这一步也可以在 `make` 之后手动操作）。这个时候调用 `nginx -V` 命令我们可以看到编译参数已经包含了新的内容。

然后通过向 master 进程发送 SIGUSR2 信号来重启 nginx 服务。

	kill -s SIGUSR2 <nginx master pid>

### 配置上传模块

直接用的官方的配置例子，除了 upload_store 做了改动。 

	server {
	    client_max_body_size 100m;
	    listen       80;
	
	    # Upload form should be submitted to this location
	    location /upload {
	        # Pass altered request body to this location
	        upload_pass   @test;
	
	        # Store files to this directory
	        # The directory is hashed, subdirectories 0 1 2 3 4 5 6 7 8 9 should exist
	        # upload_store /tmp 1;
			# 如果不需要把文件根据哈希值分开存放，则不需要填后面的值
			upload_store /data/upload;
	
	        # Allow uploaded files to be read only by user
	        upload_store_access user:r;
	
	        # Set specified fields in request body
	        upload_set_form_field $upload_field_name.name "$upload_file_name";
	        upload_set_form_field $upload_field_name.content_type "$upload_content_type";
	        upload_set_form_field $upload_field_name.path "$upload_tmp_path";
	
	        # Inform backend about hash and size of a file
	        upload_aggregate_form_field "$upload_field_name.md5" "$upload_file_md5";
	        upload_aggregate_form_field "$upload_field_name.size" "$upload_file_size";
	
	        upload_pass_form_field "^submit$|^description$";
	
	        upload_cleanup 400 404 499 500-505;
	    }
	
	    # Pass altered request body to a backend
	    location @test {
	        proxy_pass   http://localhost:8080;
			# 如果不需要传到后台操作，可以直接 return
			# return 200 "Upload success!";
	    }
	}

这个配置会把所有通过表单上传的文件保存到临时目录，然后根绝 `upload_set_form_field` 参数生成新的表单值，传送给后端处理。比如下面三条配置：
			
	upload_set_form_field $upload_field_name.name "$upload_file_name";
	upload_set_form_field $upload_field_name.content_type "$upload_content_type";
	upload_set_form_field $upload_field_name.path "$upload_tmp_path";

如果你通过页面的 `<input type="file" name="picture">` 上传了一张叫`a.jpg`的图片，那么发给后端的表单中就会有类似这样的信息：

	picture.name=a.jpg
    picture.content_type=image/jpeg
	picture.path=/data/upload/0000000001


### 搭个测试后端

用 python 的 flask 框架搭了个测试后端，json 序列化后直接原样返回表单中的数据。 

	import flask
	import json
	
	from flask import request
	
	app = flask.Flask(__name__)
	
	@app.route("/upload", methods=["POST"])
	def upload():
	    return json.dumps(request.form)
	
	if __name__ == '__main__':
	    app.run(port=8080)

然后用 `curl` 做个测试：

	curl -X POST -F testFile1=@out.json -F testFile2=@mysql.txt -F testStr=hello http://localhost:8020/upload

我们用表单发送了两个文件和一个文本消息，返回的内容是：
	
	{
		"testFile1.md5": "b6672090cd99478a4ea5a1f23a4736cd",
		"testFile1.content_type": "application/octet-stream",
		"testFile1.path": "/data/upload/0000000336",
		"testFile1.name": "out.json",
		"testFile1.size": "59930",
		"testFile2.name": "mysql.txt",
		"testFile2.size": "135582",
		"testFile2.content_type": "text/plain",
		"testFile2.md5": "d54ee6b3fa1e377b101c5b33a101aece",
		"testFile2.path": "/data/upload/0000000337"
	}

可以看到两个文件的信息都正确返回了，但是非文件的内容（ `testStr`）被忽略了。

### 其它

不通过表单形式直接发送单个文件需要开启 `upload_resumable` 参数，并且头消息里带上 `Content-Disposition` 信息。否则会提示 `415 Unsupported Media Type`。从 nginx 错误日志里可以看到： `Content-Type is not multipart/form-data and resumable uploads are off: application/x-www-form-urlencoded`

`upload_resumable` 参数是为了开启断点续传功能，这篇先不做尝试。

upload 模块不支持根据上传文件名字自动命名功能，详情可见 [Change names of temporary files to original. · Issue #69 · fdintino/nginx-upload-module](https://github.com/fdintino/nginx-upload-module/issues/69)。

所以如果要修改文件名字，建议在后端进行操作，或者修改 upload 模块，加入自己需要的功能。

