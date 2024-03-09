---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 在 MacOS 上使用 VSCode 调试 MySQL
slug: use-vscode-debug-mysql-on-macos-1j9g8z
url: /post/use-vscode-debug-mysql-on-macos-1j9g8z.html
date: '2024-02-19 12:44:53'
lastmod: '2024-02-24 11:06:43'
toc: true
tags:
  - MySQL
categories:
  - 数据库
isCJKLanguage: true
---

本文介绍在 MacOS 上使用 VSCode 调试 MySQL 的相关配置。

<!--more-->

## 依赖

```shell
$ brew install make
$ make --version
GNU Make 3.81
```

```shell
$ brew install cmake
$ cmake --version
cmake version 3.25.2
```

```shell
$ brew install openssl@1.1
$ echo export PATH="/opt/homebrew/opt/openssl@1.1/bin:$PATH" >> ~/.zshrc
$ echo export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib" >> ~/.zshrc
$ echo export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include" >> ~/.zshrc
$ echo export PKG_CONFIG_PATH="/opt/homebrew/opt/openssl@1.1/lib/pkgconfig" >> ~/.zshrc
```

```shell
$ brew install bison
$ echo export PATH="/opt/homebrew/opt/bison/bin:$PATH" >> ~/.zshrc
$ echo export LDFLAGS="-L/opt/homebrew/opt/bison/lib" >> ~/.zshrc
```

## VSCode 插件

* C/C++
* CMake tools
* CodeLLDB

## 源码

```shell
$ git clone -b 8.0 --depth=1 https://github.com/mysql/mysql-server.git
```

## 配置

切换到项目根目录：

```shell
$ cd mysql-server
```

### 调试目录

```shell
$ mkdir -p cmake-build-debug/{data,etc}
```

MySQL 配置和数据、编译产物等统一放到 `cmake-build-debug`​ 中，方便管理。

### CMake 配置

​`.vscode/settings.json`​：

```json
{
	"cmake.buildBeforeRun": true,
	"cmake.buildDirectory": "${workspaceFolder}/cmake-build-debug/build",
	"cmake.configureSettings": {
		"WITH_DEBUG": "1",
		"CMAKE_INSTALL_PREFIX": "${workspaceFolder}/cmake-build-debug",
		"MYSQL_DATADIR": "${workspaceFolder}/cmake-build-debug/data",
		"SYSCONFDIR": "${workspaceFolder}/cmake-build-debug/etc",
		"MYSQL_TCP_PORT": "3307",
		"MYSQL_UNIX_ADDR": "${workspaceFolder}/cmake-build-debug/data/mysql-debug.sock",
		"WITH_BOOST": "${workspaceFolder}/cmake-build-debug/boost",
		"DOWNLOAD_BOOST": "1",
		"DOWNLOAD_BOOST_TIMEOUT": "600"
	}
}
```

​`cmd+shift+p`​ -> `CMake: Configure`​，看到如下日志，则配置成功：

```shell
...
[cmake] -- Configuring done (1.8s)
[cmake] -- Generating done (1.3s)
[cmake] -- Build files have been written to: /Users/ruifeng/Repos/Github/mysql-server/cmake-build-debug/buil
```

## 编译

​`cmd+shift+p`​ -> `CMake: Build Target`​ -> `mysqld`​，看到如下日志，则编译成功：

```shell
[main] Building folder: mysql-server mysqld
[build] Starting build
[proc] Executing command: /opt/homebrew/bin/cmake --build /Users/ruifeng/Repos/Github/mysql-server/cmake-build-debug/build --config Debug --target mysqld --
...
[driver] Build completed: 00:10:21.425
[build] Build finished with exit code 
```

## 调试

### MySQL 配置

```shell
$ cd cmake-build-debug
$ cat > etc/my.cnf << EOF
[mysqld]
port=3307
socket=mysql.sock
innodb_file_per_table=1
EOF

# 不需要指定配置文件，因为已在 CMake 的 SYSCONFDIR 配置了。
# 也可以手动指定 build/bin/mysqld --defaults-file=etc/my.cnf --initialize-insecure
$ build/bin/mysqld --initialize-insecure
```

### 调试参数配置

​`.vscode/launch.json`​：

```json
{
	// 使用 IntelliSense 了解相关属性。 
	// 悬停以查看现有属性的描述。
	// 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
	"version": "0.2.0",
	"configurations": [
		{
			"type": "lldb",
			"request": "launch",
			"name": "Debug mysqld",
			"program": "${workspaceFolder}/cmake-build-debug/build/bin/mysqld",
			"args": [
				"--defaults-file=${workspaceFolder}/cmake-build-debug/etc/my.cnf"
			],
			"cwd": "${workspaceFolder}"
		},
		{
			"type": "lldb",
			"request": "launch",
			"name": "Debug mysql",
			"program": "${workspaceFolder}/cmake-build-debug/build/client/mysql",
			"args": [
				"-uroot",
				"-P3307",
				"-h127.0.0.1"
			],
			"cwd": "${workspaceFolder}"
		}
	]
}
```

### 启动调试

​![image](http://127.0.0.1:6806/assets/image-20240224104822-98gd59a.png)​

## 参考资料

* [MySQL 源码阅读](https://shockerli.net/tags/mysql%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/)
* [2.8.2 Source Installation Prerequisites](https://dev.mysql.com/doc/refman/8.0/en/source-installation-prerequisites.html)
