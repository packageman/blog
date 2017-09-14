---
title: 前端学习笔记
date: 2017-09-14 17:03:14
tags:
  - JavaScript
  - Html
---

- 使用以下方式可以解决使用 input type=“file” 上传图片不能连续上传多次的问题：

```js
$(document).delegate('.upload', 'change', onchangeCallback);
```

- 使用表单上传图片方法

```js
var file = files[0];
var formData = new FormData();
var fileName = uuid() + file.name.slice(file.name.lastIndexOf('.'));
formData.append("key", fileName);
formData.append("policy", config.policy);
formData.append("OSSAccessKeyId", config.accessKeyId);
formData.append("signature", config.signature);
formData.append("success_action_status", 200);
formData.append("file", file);

var request = new XMLHttpRequest();
request.open("POST", config.uploadDomain, true);
request.onreadystatechange = function () {
  if(request.readyState === 4 && request.status === 200) {
    fileInput.parent().parent().find('input[name="message"]').attr('value', config.domain + "/" + fileName);
    fileInput.parent().parent().parent().submit();
  }
};

request.send(formData);
```
