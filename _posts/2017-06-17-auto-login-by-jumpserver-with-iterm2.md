---
layout: post
title:  "iTerm2 经 Jumpserver 自动登录服务器"
date:   2017-06-17 01:30:40 +0800
categories: tools
---

- 首先配置 .ssh/config

```
Host jumpserverhost
Hostname 172.16.0.8                # your Jumpserver IP
User username                      # login name for jumpserver
Port 80                            # jumpserver port
IdentityFile ~/.ssh/jumpserver.pem # jumpserver pem downloading from jumpserverweb
```

- 编写登陆脚本 jump, 务必把 jumpserverhost 替换为自己的登陆地址

```
#!/usr/bin/expect
if { $argc != 2 } {
    send_user "Usage: host user\n"
    exit
}
set HOST [lindex $argv 0]
set USER [lindex $argv 1]

# login jumpserver, use your own user@hostname to replace jumpserverhost
catch { spawn ssh jumpserverhost }
expect {
    "*Opt or ID*"
    # p list server, 0 select #0 server
    { send "p\n0\n" }
}
# login real server
expect {
    "*$ "
    { send "ssh -l $USER $HOST\n" }
}
# enter interactive mode
interact
```

- 设置自动登录脚本的属性, 并把文件拷贝到 PATH 路径下: `chmod +x jump && cp jump /usr/bin/`
- 打开 iTerm2, 用 <kbd>⌘</kbd>+<kbd>,</kbd> 打开 Preferences, 选择 Profiles 页, 为主机 `10.8.8.8` 添加一个新 Profile. 务必给 jump 命令提供两个参数: `hostname username`
![](/assets/images/jump-profile.png)
- 再选择 Profiles 页的 Advanced, 配置密码触发器. 由于使用 SSH 登录, 所以配置中的正则表达式为密码输入的提示符 `password:`
![](/assets/images/jump-trigger.png)
- 用 <kbd>Alt</kbd>+<kbd>⌘</kbd>+<kbd>F</kbd> 打开密码管理器 Password Manager, 添加主机 `10.8.8.8` 的账号与密码
![](/assets/images/jump-password.png)
- 上面的配置完成后, 即可使用 <kbd>⌘</kbd>+<kbd>O</kbd> 打开 Profiles, 选择 Example 即可通过 Jumpserver 登录到 `10.8.8.8`
