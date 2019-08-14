# 分支
master(博客内容)  
hexo(所有网站文件)

# 在新的电脑上搭建
1. npm install hexo
2. hexo init
3. npm install
4. npm install hexo-deployer-git(hexo分支)
5. 修改_config.yml中的deploy参数(分支应为master)
6. git add .
7. git commit -m "..."
8. git push origin hexo
9. hexo g -d(部署到github也就是更新master代码)

# 日常修改流程
1. git add .
2. git commit -m "..."
3. git push origin hexo
4. hexo g -d(部署到github也就是更新master代码)

# 常用操作
$ hexo n "博客名称"  => hexo new "博客名称"   #这两个都是创建新文章，前者是简写模式  
$ hexo p  => hexo publish  
$ hexo g  => hexo generate  #生成  
$ hexo s  => hexo server  #启动服务预览  
$ hexo d  => hexo deploy  #部署  

$ hexo server   #Hexo 会监视文件变动并自动更新，无须重启服务器。  
$ hexo server -s   #静态模式  
$ hexo server -p 5000   #更改端口  
$ hexo server -i 192.168.1.1   #自定义IP  
$ hexo clean   #清除缓存，网页正常情况下可以忽略此条命令  
$ hexo g   #生成静态网页  
$ hexo d   #开始部署  
