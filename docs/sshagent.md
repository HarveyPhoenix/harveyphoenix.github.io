## ssh-agent 多会话共享解决方案

### 问题

在使用wsl的时候发现，在一个窗口启动`ssh-agent`并添加密钥后，新开一个窗口就无法继续使用该agent和密钥了。

```bash
# 单机器上建立ssh-agent的方法

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa*
```

### 问题的解决方案

在文件`~/.bashrc`中添加下面内容：

```bash
if ! pgrep -u "$USER" ssh-agent > /dev/null; then
    ssh-agent -s > "/home/$USER/ssh-agent.env"
fi
if [[ ! "$SSH_AUTH_SOCK" ]]; then
    source "/home/$USER/ssh-agent.env" >/dev/null
fi
```


### 参考

- [1] [SSH 密钥](https://wiki.archlinuxcn.org/zh-hans/SSH_%E5%AF%86%E9%92%A5#ssh-agent)