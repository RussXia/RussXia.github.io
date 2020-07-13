---
layout: blog
title: "给GitHub Pages配置域名和https"
catalog: true
tag: [杂项,2019]
---

# 给GitHub Pages配置域名和https

一直都想买个域名供自己的博客使用，最近终于下定决心剁手了。在这里记录下给GitHub Pages配置域名和https的过程，以供参考。

可以参考github上的文档: [https://help.github.com/en/github/working-with-github-pages/configuring-a-custom-domain-for-your-github-pages-site](https://help.github.com/en/github/working-with-github-pages/configuring-a-custom-domain-for-your-github-pages-site)



## 申请域名，配置解析

首先，你要申请一个域名。域名服务的提供商有很多，国内的如阿里的万网，腾讯云。国外的如全球最大的Godaddy等等。在这里你要选择一家提供商，然后买下你的域名。

这里我选择的是腾讯云(<del>因为最近打折</del>)，国内申请域名是要实名注册登记的。

+ 在完成登记流程后，将你的域名解析配置到你的 `your_user_name.github.io` 上去。这里你也可以指定A类IPv4的方式解析到你的GitHub Pages上，但是不推荐，容易被挟持。

![image-20191206145549136](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/domain_cloud_setting.png)

+ 然后在你的GitHub项目的settings中，找到Custom domain，配置为你购买的域名。GitHub会自动为你创建一个CNAME文件，内容就是你所配置的域名，勾选上 `Enfore HTTPS` 选项。

  ![image-20191206145943688](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/domain_github_setting.png)

到此，你就可以通过你的域名访问你的GitHub博客了。

## 题外话

GitHub 和 Let’s Encrypt 合作，我们只需要勾选了 `Enfore HTTPS` ，就可以使用 Let’s Encrypt 签发的 SSL 证书。

![image-20191206150559273](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/domain_https_setting.png)