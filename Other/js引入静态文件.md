---
title: js引入静态文件
date: 2019-05-16 19:34:00
categories: node相关
summary: 有点水……
---

# Vue项目中js引入静态文件

需求：前台接收头像数据数据，但头像可能是空，如果是空，就引入本地项目中的一张图片，代码：

```javascript
let that = this;
api.getAvatar(id).then(res => {
    let data = res.data.data;
    if (data == "") {        //如果数据为“”
        data = "../../assets/img/img.jpg";        
    }
    that.avatar = data;
});
```

页面：

```vue
<div class="user-avator">
    <img  :src="avatar">
</div>
```

但是问题是图片没有出来，网页查看源代码

```html
<img data-v-1f234af1="" src="../../assets/img/img.jpg">
```

发现赋值成功，理论还上没有问题，于是我将页面代码改为这样试试看：

```vue
<div class="user-avator">
    <img  src="../../assets/img/img.jpg">
</div>
```

发现图片出来了，很是奇怪，于是再次网页查看代码：

```html
<img data-v-1f234af1="" src="img/img.146655c9.jpg">
```

发现src并不是直接显示的“../../assets/img/img.jpg”，于是想到webpack在打包这个项目的时候会将这些静态文件进行整理（压缩之类的操作），并不是直接显示的。

第一个解决办法，该图片不要在项目存储，可以放到服务器上，通过链接显示。

第二个解决办法，突然想到可以在js中引入静态资源，和引入js模块没有区别。

```javascript
const pic = require("../../assets/img/img.jpg");  //引入图片模块

***************************************************
let that = this;
api.getAvatar(id).then(res => {
    let data = res.data.data;
    if (data == "") {        
        data = pic;        
    }
    that.avatar = data;
});
```

自己总结了一下，地址写在html中，会有相关的插件和编译器来处理，所以有效果，在js中直接写字符串，不能够被解析，但是通过模块引入的话，就能够被编译器处理。

webpack还是要好好学习的。



# vue-cropper文件上传

插件github地址：https://github.com/xyxiao001/vue-cropper



由于插件是拿的别人的代码过来的，所以没什么特别记录的（实际上是有点看不懂…………），主要是记录是上传到后端（java）：

```javascript
let that = this;
let param = new FormData();
that.upload = true;
param.append("id", that.id);
//通过append向form对象添加数据
param.append("file", that.file);
let config = {
    //添加请求头
    headers: { "Content-Type": "multipart/form-data" },
    //添加上传进度监听事件
};
let { data } = await api.uploadAvater(param, config);
***************************************************
//商家上传头像
async function uploadAvater(param,config) {
   return await axios.post(url+"shop/uploadAva", param, config);
}
```

以后有新的知识了再过来记录吧！

# v-viewer插件

插件github地址:https://github.com/mirari/v-viewer

简单的使用，首先要在main.js中引入：

```javascript
import Viewer from 'v-viewer';
import 'viewerjs/dist/viewer.css';
Vue.use(Viewer, {
    defaultOptions: {
      zIndex: 9999
    }
})
```

使用的时候直接使用组件

```vue
<viewer :images="photo">
    <img v-for="(src,index) in photo"
         style="cursor:pointer;width: 200px; height: 200px"
         :src="src"
         :key="index"
         >
</viewer>
```

注意viewer标签必须要有images属性，它是一个数组，如果只想要显示一张图片，也是放到数组里面（只有一个元素的数组），否则会出现意料之外的错误。

确实挺简单的，详细可以查看文档！