### 个人网站
1. hexo + next
2. next 有两个origin my_origin 是自定义next配置
```
my_origin       git@github.com:Xuwudong/myNext.git (fetch)
my_origin       git@github.com:Xuwudong/myNext.git (push)
origin  https://github.com/next-theme/hexo-theme-next (fetch)
origin  https://github.com/next-theme/hexo-theme-next (push)
```

#### 开发方式
1. npx hexo clean
2. 本地测试： npx hexo server --debug
3. 部署： npx hexo g -d

#### 问题
1. 怎么解决新发布的文章提示 "未找到相关的Issues进行评论"

A：需要在最底部的文章登录一下自己的github账号，然后系统会自动在git_talk repo项目中创建一个新的issue实现评论
