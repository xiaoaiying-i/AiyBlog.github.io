```
# 登录  输入命令后会提示输入密码
# 如：sftp -P 7022 test_acc_sftp@127.0.0.1
sftp user@ip


# 帮助  登录后不清楚相关命令可用help查看
help
  
# 目录
pwd和lpwd

# 目录列表
ls和lls


# 上传 把linux当前目录下的a.txt文件上传到sftp服务器的当前目录下
put a.txt

#下载 把sftp服务器当前目录下的b.txt文件下载到linux当前目录下
get a.txt

# 退出
exit和quit


# 这个是指在linux上执行command这个命令， 比如!ls是列举linux当前目录下的东东， !rm a.txt是删除linux当前目录下的a.txt文件
# 在sftp> 后输入命令， 默认值针对sftp服务器的， 所以执行rm a.txt删除的是sftp服务器上的a.txt文件， 而非本地的linux上的a.txt文件
# 如：!echo testtest > test.txt
!command
```

> 命令前加l相当于local， 命令作用于当前服务器，  不加作用于sftp服务器