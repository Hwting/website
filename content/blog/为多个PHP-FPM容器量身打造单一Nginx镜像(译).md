
---
title: "为多个PHP-FPM容器量身打造单一Nginx镜像(译)"
date: 2018-06-06T16:00:00+08:00
draft: false
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
banner: "https://i.imgur.com/XoCNwnk.jpg"
summary: "这篇博客主要讲述了如何创建一个可以关联docker环境变量与nginx配置文件的nginx镜像，供你所有的`PHP-FPM`容器应用。"
tags: ["docker","PHP","Nginx"]
categories: ["docker"]
---

# 为多个PHP-FPM容器量身打造单一Nginx镜像

译者的话：这篇博客主要讲述了如何创建一个可以关联docker环境变量与nginx配置文件的nginx镜像，供你所有的`PHP-FPM`容器应用。

最近我一直在努力部署一套使用Docker容器的PHP微服务。其中一个问题是我们的PHP应用程序被设置为与`PHP-FPM`和`Nginx`一起工作（而不是这里所说的简单的[Apache/PHP](https://www.shiphp.com/blog/2017/php-web-app-in-docker)设置）,因此每个PHP微服务需要两个容器（也就是相当于两个Docker镜像）：
* PHP-FPM容器
* Nginx容器

假设一个应用运行超过六个PHP微服务，算上你的dev和prod环境，那么最终差不多会产生接近30个容器。我决定构建一个单独的Nginx Docker镜像，将`PHP-FPM`主机名作为环境变量映射到这个镜像里面独特的配置文件中，而不是为每个`PHP-FPM`微服务的镜像构建独特的Nginx镜像。

![](https://i.imgur.com/XoCNwnk.jpg)


在这篇博客文章中，我将概述我从上述方法1到方法2的过程，最后用介绍如何使用新定制Nginx Docker镜像的解决方案来结束这篇博客。

我已经将这个镜像开源[Github](https://github.com/shiphp/nginx-env)，所以如果这刚好是您经常遇到的问题，请随时查看。

## 为什么是Nginx？

`PHP-FPM`和Nginx一起使用可以产生更好的PHP应用程序性能，但缺点是PHP-FPM Docker镜像默认没有像`PHP Apache`镜像那样与Nginx捆绑在一起。
如果您想将Nginx容器连接到PHP-FPM后端，则需要将该后端的DNS记录添加到您的Nginx配置中。

例如，如果PHP-FPM容器作为名为`php-fpm-api`的容器运行，那么您的Nginx配置文件应该这样写：

```nginx
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        # This line passes requests through to the PHP-FPM container
        fastcgi_pass php-fpm-api:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
```

如果你只服务一个PHP-FPM容器应用，在你的Nginx容器的配置文件中硬编码对应的名字是可以的。但是，如我上面提到的，每个PHP服务都需要一个对应的Nginx容器，我们就需要运行多个Nginx容器。创建一个新的Nginx镜像（我们后面必须维护和升级）将是一件痛苦的事情，因为即使管理一堆不同的卷，对于更改单个变量名称似乎也有很多工作要做。

## 第一个解决方案：使用Docker文档里提到的方法`envsubst`

起初，我认为这很容易。在Docker文档中关于如何使用`envsubst`有一个很好的[小章节](https://docs.docker.com/samples/library/nginx/#using-environment-variables-in-nginx-configuration)，但不幸的是，这不适用于我的Nginx配置文件：

**vhost.conf**
```nginx
server {
    listen 80;
    index index.php index.html;
    root /var/www/public;
    client_max_body_size 32M;

    location / {
        try_files $uri /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass ${NGINX_HOST}:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```
我的`vhost.conf`文件用到了好几个Nginx内置的环境变量，结果当我运行docker文档里提到的如下命令行时，提示错误：`$uri`和`fastcgi_script_name`未定义。

```shell
/bin/bash -c "envsubst < /etc/nginx/conf.d/mysite.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```
这些变量通常由Nginx本身传入，所以不容易搞清楚他们是什么和怎么进行参数传递的，而且这会影响容器的动态可配置性。

## 另一个差点成功的docker镜像

接下来，我开始搜索不同的Nginx的基础镜像。找到了两个，但是这两个都是两年没有更新了。我从[martin/nginx](https://hub.docker.com/r/martin/nginx/)开始，尝试看看能不能得到一个可以工作的原型。
Martin的镜像有点不太一样，因为它要求特定的文件目录结构。我先在`Dockerfile`中添加了：

```
FROM martin/nginx
```
接下来，我添加了`app/`空目录，只包含一个`vhost.conf`文件的`conf/`目录。

**vhost.conf**

```nginx
server {
    listen 80;
    index index.php index.html;
    root /var/www/public;
    client_max_body_size 32M;

    location / {
        try_files $uri /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass $ENV{"NGINX_HOST"}:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

这个跟我原始的配置文件差不多，只修改了一行：`fastcgi_pass $ENV{"NGINX_HOST"}:9000;`。现在当我想要启动一个Nginx容器和一个叫`php-fpm-api`的PHP容器的时候，我可以先编译一个新的镜像，然后在它运行的时候传递给它对应的环境变量：

```shell
docker build -t shiphp/nginx-env:test .
docker run -it --rm -e NGINX_HOST=php-fpm-api shiphp/nginx-env:test
```

成功了！但是，这个方法有两个问题困扰着我：
1. 基础镜像版本陈旧，两年多没更新了。这可能会造成安全和性能风险。
2. 要求一个`app`的空目录似乎没啥必要，再加上我的文件放在不同的目录。

## 最终解决方案

我觉得Martin的镜像是个不错的自定义方案选择。所以，我`fork`了他的仓库并构建了一个[新的并解决了以上两个问题的Nginx基础镜像](https://hub.docker.com/r/shiphp/nginx-env/)。现在，如果你想运行一个伴随着nginx容器的动态命名后端应用，你只需要简单地这么做：

```shell
# Pull down the latest from Docker Hub
docker pull shiphp/nginx-env:latest

# Run a PHP container named "php-fpm-api"
docker run --name php-fpm-api -v $(pwd):/var/www php:fpm

# Start this NGinx container linked to the PHP-FPM container
docker run --link php-fpm-api -e NGINX_HOST=php-fpm-api shiphp/nginx-env
```

如果你想自定义这个镜像，添加你自己的文件或者Nginx配置文件，只需要像下面这样扩展你的`Dockerfile`:

```
FROM shiphp/nginx-env

ONBUILD ADD <PATH_TO_YOUR_CONFIGS> /etc/nginx/conf.d/
```
现在我所有的PHP-FPM容器都使用单个Nginx镜像的实例，当我需要升级Nginx、修改权限或者配置一些东西的时候，这让我的生活变得简单多了。

所有的代码都放在[Github](https://github.com/shiphp/nginx-env)上面了。如果您发现任何问题或想要提出改进建议，请随时创建`issue`。如果您对这个问题或Docker相关的任何问题，可以在[Twitter](https://twitter.com/shiphpnow)上找我一起讨论。

[原文链接](https://www.shiphp.com/blog/2018/nginx-php-fpm-with-env)






