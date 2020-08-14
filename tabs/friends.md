---
title: 交友平台
comments: true
---

<style>
img.avatar-img {
    width: 60pt;
    height: 60pt;
    border-style: solid;
    border-color: #454545ae;
    box-shadow: 0 0 6px 3px #111;
    border-radius: 50%;
}
img.avatar-img:hover {
    -ms-transform: scale(1.2);
    -moz-transform: scale(1.2);
    -webkit-transform: scale(1.2);
    transform: scale(1.2);
}
.post h1 {
    text-align: center;
}
.post-content {
    text-align: center;
}
</style>


## 这里是友链大家庭！我的朋友们都在这里，有空也去转一转吧。

<br/>

---

{% for item in site.data.friends %}

<a href="{{item.url}}">
<img src="{{item.avatar}}" class="avatar-img">
</a>

## [{{item.name}}]({{item.url}})  
**{{item.slogan}}**

---

{% endfor %}

<br/>

## 加入我们

如果你也想成为这里的一员，欢迎发送邮件至 [i@cyanoxygen.xyz](mailto:i@cyanoxygen.xyz) ，或在下方评论区留言。  

你需要提供：

你的昵称  
你的头像的链接（推荐使用 Gravatar）  
你的格言  
你的博客链接（其他平台的链接也可以）  

### 我的友链信息

Name: `Reliena's Garage`  
Avatar: `https://www.gravatar.com/avatar/6765191a1ca4490345e4bf19b78003a3?s=512`  
Motto: `Experience more, write more.`  
URL: `https://blog.cyanoxygen.xyz`  

## 愿这里的每一位朋友都能安好。
