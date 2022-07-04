```bash
hexo n "博客名称"  => hexo new "博客名称"   #这两个都是创建新文章，前者是简写模式
hexo p  => hexo publish
hexo g  => hexo generate  #生成
hexo s  => hexo server  #启动服务预览
hexo d  => hexo deploy  #部署  

hexo server   #Hexo 会监视文件变动并自动更新，无须重启服务器。
hexo server -s   #静态模式
hexo server -p 5000   #更改端口
hexo server -i 192.168.1.1   #自定义IP
hexo clean   #清除缓存，网页正常情况下可以忽略此条命令
hexo g   #生成静态网页
hexo d   #开始部署
```

