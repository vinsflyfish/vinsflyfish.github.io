# 博客hugo升级及操作简要

## hugo升级之后之前的参数不太兼容

1. 本地安装最新的hugo
2. 创建网站 hugo new site XXXX
3. 切换到网站目录 cd XXXX
4. 创建子目录posts下的博客文件 hugo new posts/xxxx.md 推荐采用短横线的分割方式 first-blog.md
5. hugo server 本地验证
6. 验证没问题了 hugo -D 会生成静态文件在 public目录下面，将public下的文件同步到 xxxx.github.io的git即可,建议采用cp的方式，避免删除图片

之前的图片文件位于 xxx.github.io/img/xxx.png 这部分不会覆盖

hugo主要操作是下载theme 本博客采用的是 Mainroad 然后将theme写入 config.toml 参照 Mainroad主题网站的配置，配置好config.toml

## 完整的操作说明
1. brew install hugo 实际上因为镜像源的缘故，下载不下来，主要是brew版本过低，参照教程卸载了brew 然后切换了国内的镜像源重新安装好了brew
2. brew install hugo
3. hugo new site githubpages 创建了当前工程的目录，本仓库目录即为 githubpages/下面的内容
4. 下载Mainroad主题 git clone https://github.com/vimux/mainroad.git themes/mainroad
5. config.toml 配置中加入 Mainroad的主题 theme = "mainroad"
6. 参照 Mainroad的github readme 配置了一份基础的 config.toml
7. hugo new posts/first-blog.md 创建了第一个博客，然后将之前的 md也拷贝过来(这一步非必须，可以自己手写md，要注意md的头，自动生成会带额外可用的头)
8. hugo server 本地浏览器验证效果
9. hugo -D 生成发布的静态文件 在当前public目录下，本工程中已被移除，生成的目录同步无意义
10. 拉取远端目录备份 hugo工程
11. git remote add origin git.xxxx远端的地址，本地关联上远端
12. 拷贝本地hugo工程
13. git add .
14. git commit -m "full sync"
15. git push origin master
