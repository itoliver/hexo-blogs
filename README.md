部署步骤
1. 安装nodejs
  * 先下载安装nodejs
  [nodejs for windows](https://nodejs.org/dist/v8.11.1/node-v8.11.1-x64.msi)

2. 安装git
  * 下载安装git bash
  [git for windows](https://github-production-release-asset-2e65be.s3.amazonaws.com/23216272/caadf4ec-1641-11e8-8f85-577fa933ab56?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20180410%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20180410T065906Z&X-Amz-Expires=300&X-Amz-Signature=71d49d50ad078ce3223a980361812952314ab8854b85f5df339ec71fa459f09d&X-Amz-SignedHeaders=host&actor_id=37535883&response-content-disposition=attachment%3B%20filename%3DGit-2.16.2-64-bit.exe&response-content-type=application%2Foctet-stream)

3. 启动git bash，创建目录
  * mkdir /blogs

4. 安装hexo
  * npm install hexo -g
  * hexo -v

5. git clone https://github.com/itoliver/hexo-blogs.git /blogs
6. git clone https://github.com/itoliver/next.git /blogs/themes/next
7. 记得修改.git的仓库地址
  * cd /blgos
  * 在生成以及部署文章之前，需要安装一个扩展：
  npm install hexo-deployer-git --save
8. hexo g
9. hexo s
##大致的思路就这样，具体自己把握
