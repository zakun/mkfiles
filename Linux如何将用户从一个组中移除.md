## Linux如何将用户从一个组中移除


> gpasswd -d userName groupName

### id用来查看用户属性

```
[root@gl gl]# id root
uid=0(root) gid=0(root) groups=0(root),1000(gl)
[root@gl gl]# gpasswd -d root gl
Removing user root from group gl
[root@gl gl]# id root
uid=0(root) gid=0(root) groups=0(root)
```

