---
layout: default
title: "在本地运行 Jekyll"
date: 2018-08-02 12:38:00
published: 2018-08-02 12:38:00
comments: true
categories: it
tags: [it]
author: ykfq
---


<p>
Jekyll 是一个使用 ruby 写的静态网站生成器，特别适合发布 github pages。  
我们以使用 `minimal-mistakes` 这个主题为例，在本地启动一个 jekyll 博客。
</p>

<!--more-->


## 安装 rvm

RVM 是一个命令行工具，可以提供一个便捷的多版本 Ruby 环境的管理和切换。

### 设置 yum源  

由于国内网络问题，我们选择使用中国科技大学的 yum 镜像源，配置不赘述。
当已经指定了yum镜像地址，就可以禁用yum的`fastestmirror`插件，跳过镜像源检测，直接安装：
```bash
sed -e 's!enabled=1!enabled=0!g' -i /etc/yum/pluginconf.d/fastestmirror.conf
```

### 安装依赖  
```bash
yum -y install gcc-c++ patch readline readline-devel zlib zlib-devel \
                libyaml-devel libffi-devel openssl-devel make bzip2 \
                autoconf automake libtool bison iconv-devel sqlite-devel
```

### 安装 rvm  

安装 rvm 时需要下载一些依赖包，建议先设置代理：
```bash
gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

export https_proxy="http:192.168.xx.xx:443"
curl -sSL https://rvm.io/mpapis.asc | gpg --import -
curl -L get.rvm.io | bash -s stable

source /etc/profile.d/rvm.sh
rvm reload
rvm requirements run
```


## 安装 ruby

先设置 ruby 的国内镜像源，再使用rvm安装ruby，我这里选择了淘宝：

```bash
sed -i -E 's!https?://cache.ruby-lang.org!https://cache.ruby-china.com!' /usr/local/rvm/config/db
```
~~由于Jekyll依赖`ffi`这个模块，它的当前最新版本是 `ffi 1.9.18`, 支持 Ruby 的版本要求 < 2.5, >= 2.0，因此我选择安装ruby 2.4:~~  
ffi 从 1.9.21 版开始支持 ruby 2.5, 同时放弃了支持 ruby 1.8.7，安装高版本 ruby 即可。

```bash
rvm list known
rvm install 2.6.3
rvm list

#设置默认ruby
rvm use 2.6.3 --default
ruby --version
```


## 安装 Jekyll

### 设置 gem mirrors

Gem 是 ruby 的模块管理工具。  
gem 安装 jekyll 时会安装较多依赖，我们依然选择淘宝镜像：

- 全局设置
  ```bash
  gem sources -l
  gem sources --remove https://rubygems.org --add https://gems.ruby-china.com
  ```

- 当前项目
  ```bash
  bundle config mirror.https://rubygems.org https://gems.ruby-china.com
  
  # 或者修改项目目录下的`Gemfile`文件：
  vim Gemfile
  #source 'https://rubygems.org'
  source 'https://gems.ruby-china.com'
  ```

### 安装 Jekyll

```bash
gem install bundler jekyll
jekyll new my-awesome-site
cd my-awesome-site
```


## 使用 minimal-mistakes 主题

有两种方式使用 minimal-mistakes 主题：
- 使用 jekyll 新建一个项目并在项目的 Gemfile 中引用主题
- clone 该主题并基于该主题直接修改

### 在项目中引用该主题
  ```bash
  echo gem "minimal-mistakes-jekyll" >> Gemfile
  echo theme: minimal-mistakes-jekyll >> _config.yml
  #安装依赖
  bundle install
  ```

### clone 主题并修改
  ```
  git clone https://github.com/mmistakes/minimal-mistakes.git
  cd minimal-mistakes && git checkout tags/4.13.0
  rm -rf .editorconfig .gitattributes .github docs test CHANGELOG.md minimal-mistakes-jekyll.gemspec README.md screenshot-layouts.png screenshot.png
  
  vim Gemfile
  source 'https://gems.ruby-china.com'
  gem "minimal-mistakes-jekyll"
  
  #更新依赖
  bundle update
  ```

### 启动服务
  ```bash
  bundle exec jekyll serve --host 0.0.0.0 -w
  ```

### 访问网站

[http://192.168.100.100:4000](http://192.168.100.100:4000)


## 参考文档

- [https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide)