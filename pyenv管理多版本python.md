---
title: CentOS 7 上 pyenv 实现python多版本管理
date: 2018-02-06 14:30:00
tags: 
  - python 
categories: 
  - linux  
---

- 获取pyenv `git clone https://github.com/pyenv/pyenv.git ~/.pyenv`

- 设置环境变量

  ```bash
  echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
  echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
  echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
  ```

- 重启当前shell `exec "$SHELL"`

- 安装依赖，防止安装其他版本的Python时报错

  ```bash
  yum install gcc gcc-c++ readline readline-devel readline-static -y
  yum install openssl openssl-devel openssl-static -y
  yum install sqlite-devel -y
  yum install bzip2-devel bzip2-libs -y
  ```

- 查看当前系统默认Python版本命令 `python --version`

- pyenv的用法以及部分相关命令详解

  ```txt
  pyenv 命令使用规则如下：pyenv <command> [<args>]

  //查看可安装的版本列表
  pyenv install -list
  //安装指定的Python版本,例如3.6.4
  pyenv install 3.6.4
  //切换当前目录的版本号为3.6.4
  pyenv local 3.6.4
  //切换全局目录Python版本号为3.6.4
  pyenv global 3.6.4
  //切换全局目录Python版本号系统默认版本
  pyenv global system
  //刷新shims
  pyenv rehash

  ```
- pyenv的部分命令说明

  ```txt
  commands      列出所有pyenv可用命令
  local         设置或列出当前环境下Python的版本号
  global        设置或列出全局环境下Python的版本号
  shell         设置或列出Shell环境下Python的版本号
  install       安装指定版本的Python
  uninstall     卸载指定版本的Python
  rehash        重新加载pyenv的shims路径，安装完python版本之后执行该命令
  version       展示当前python版本号及其路径
  versions      列出所有pyenv已安装的并且可用的Python版本
  ```
  其他命令可通过`pyenv --help`获取帮助文档，`pyenv help <command>`可以获取指定命令的详细用法.

以上设置仅生效于当前操作用户，其他用户无效。