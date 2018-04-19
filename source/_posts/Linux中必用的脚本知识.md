---
title: Linux中必用的脚本知识
date: 2018-03-22 21:35:37
tags:
    - Linux
    - Script
categories:
    - Linux Basic
---

## 一、如何打出花样日志？
平时我们用得最多的log打印命令echo都是直接用，却很少用它的-e参数，其实合理的使用-e参数可以从视觉效果上就区分出log的等级。
```
echo -e "\e[31;40;5muse echo colorfully\e[0m" 
```
以上命令打出日志“use echo colorfully”为红色字体（31），黑色背景（40），闪烁显示（5）。下面是具体的配置参数表：

* 字体色: default=0,[30-37] = [黑色，红色，绿色，黄色，蓝色，洋红，青色，白色]
* 背景色: default=0,[40-47] = [黑色，红色，绿色，黄色，蓝色，洋红，青色，白色]
* 0 关闭所有属性、1 粗体、4 下划线、5 闪烁、7 反显
以上各参数均可以同时使用，但是颜色重复使用时以最后一个为准。
<!-- more -->

## 二、你一定要用起来的两个文件

1. `/home/username/.bash_profile`
登陆就会自动运行的文件。我们可以把每次登陆都需要做的事写个脚本然后加到这个文件中，比如配置开发环境。这样就省去了每次登陆都要自己手动配置的麻烦。

2. `/home/username/.bash_rc`
这是bash的配置文件，可以定义bash提示符还有自己的简易命令。
```
# PS1定义了bash提示符格式 , 下面配置的格式为@host currentPath > 
PS1="\[\e[31;1m\]@\h \[\e[0m\]\W \[\e[31;1m\]> \[\e[0m\]"
# 可以给经常使用的较长的命令设置一个简化命令
alias shortcmd='yourscript.py action -param1 -param2'
```

## 三、学会操作字符串

1. `#` 号截取，删除匹配字符及左边字符
`#` 删除从左往右第一个匹配的字符以及它左边的字符
`##` 删除从左往右最后一个匹配字符以及它左边的字符
```
string="home/testUser/testfolder"

new_String1=${string#*/}  #"testUser/testfolder"
new_String2=${string##*/} #"testfolder"
```

2. `%` 号截取，删除匹配字符及右边字符
`%` 删除从左往右第一个匹配的字符以及它右边的字符
`%%` 删除从左往右最后一个匹配的字符以及它右边的字符
```
new_String3=${string%/*}    #"home"
new_String4=${string%%/*}   #"home/testUser"
```

3. 字符编号截取  `${string:N1:N2}`
N1 : 从第几个字符开始，>= 0 从左边开始，< 0 (0-N)从右边开始
N2 : 需要截取的字符个数， 缺省时为截取到字符串最后（右）
```
new_String5=${string:2:2}   #"me"
new_String6=${string:0-6:4} #"fold"
new_String7=${string:0-6}   #"folder"
new_String8=${string:6}     #"/testUser/testfolder"
```

4. 字符拼接  `${string1}${string2}`
```
new_String9=${new_String7}${new_String8}  #"folder/testUser/testfolder"
```

## 四、通过 expect 实现自动远程登陆的模板

1.学会在 shell 脚本执行时读取密码而不是把密码直接赋值给一个变量
```
read -p "Your Passwd: " -s PASSWARD #变量PASSWARD存放输入的密码
```

2.在 shell 中插入使用 expect
```
expect<<EOF
...
#expect codes
...
EOF
```

3.expect自动登陆不退出
```
set timeout 10 #timeout is 10s
spawn ssh $USER@$SERVER
expect {
"(yes/no)?"
{
        send "yes\n"
        expect "*assword:" { send "$PASSWARD"}
}
"*assword:"
{
        send "$PASSWARD"
}
}
interact
```

4.expect自动登陆执行指定命令并退出
```
spawn ssh $USER@$SERVER
expect {
"(yes/no)?"
{
       send "yes\n"
       expect "*assword:" { send "$PASSWARD"}
}
"*assword:"
{
       send "$PASSWARD"
}
}

expect "# "
send "$YOUR_CMD\n"
expect "$EXPECT_PRINTS"
send "exit\n"
expect eof
```