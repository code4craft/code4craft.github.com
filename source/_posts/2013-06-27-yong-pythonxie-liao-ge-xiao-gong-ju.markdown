---
layout: post
title: "写了个快捷保存文本的shell工具"
date: 2013-06-27 22:14
comments: true
categories: python, shell
---
作为一个Java程序员，对脚本语言程序员自己随时写些小工具，已经羡慕很久了，于是最终开始了脚本语言的学习。

<!-- more -->

第一个工具是个助记程序。像工作中我们总会管理一些文本，比如内网服务器地址、git仓库地址什么的，之前一直都是要登录一个平台去拷贝下来，然后粘贴到shell里，特别麻烦。想写了个工具，为一个文本写一个助记符，然后保存到文件里，每次可以调用程序把文件读出来，再根据助记符查找到对应文本。用法可以这样：
	
	add alias text
	get alias

例如：
	add getter https://github.com/code4craft/getter 
	
也可以用``符号穿插到程序里：

	git clone `get getter`

刚开始用python写了一遍，把文件保存到~/.getrc，并每次读出来并更新。

后来想了想，这个方法没有自动补全，太麻烦！有没有好办法？最终想到，干脆把name作为可执行文件名，value作为程序的输出，每个name对应一个输出，再以特定前缀开始，这样子也有自动提示了，岂不是更好？于是就有了shell版本：

	add.sh
	
	 #!/bin/sh
    FILE_PATH=/usr/local/getter
    PREFIX=-
    [ -d "$FILE_PATH" ] || mkdir -p $FILE_PATH
    if [ -z "$1" ]
    then
        echo "Usage: $0 alias [text]"
    else
        FILE_NAME=$FILE_PATH/$PREFIX$1
        if [ -z "$2" ]
        then
            rm -f $FILE_NAME
        else
            echo "#!/bin/sh" > $FILE_NAME
            echo "echo \"$2\"" >> $FILE_NAME
            chmod +x $FILE_NAME
        fi
    fi

记得在/etc/bashrc中加一行
	
	export PATH=$PATH:/usr/local/getter
	
后来一想，那么如果我要保存一个可执行的shell命令，执行时是不是要`-xx`?干脆把命令直接写到脚本里，也别echo了！

	edd.sh
	
	 #!/bin/sh
    FILE_PATH=/usr/local/getter
    PREFIX=-
    [ -d "$FILE_PATH" ] || mkdir -p $FILE_PATH
    if [ -z "$1" ]
    then
        echo "Usage: $0 alias [command]"
    else
        FILE_NAME=$FILE_PATH/$PREFIX$1
        if [ -z "$2" ]
        then
            rm -f $FILE_NAME
        else
            echo "#!/bin/sh" > $FILE_NAME
            echo "$2 \$1 \$2 \$3" >> $FILE_NAME
            chmod +x $FILE_NAME
        fi
    fi

比如之后我要保存一条命令"curl http://code4craft.github.io/blackhole/install.sh | sh"，可以这么用：
	
	#保存命令
	edd install "curl http://code4craft.github.io/blackhole/install.sh | sh"
	#执行命令
	-install

代码已经放到了github:

[https://github.com/code4craft/getter](https://github.com/code4craft/getter)