# [基于github issues记录博客](https://github.com/void-syh/blog/issues/1)

## 前言
最近闲暇时间决定维护一下自己很久没有维护的github（大学的时候弄的一团糟，苏米马赛），浏览了几个大佬的git，发现记博客和一些随笔的时候，大家都喜欢用issue去记录。
issue记录有几个好处
- 本身支持markdown编辑，写起来快速方便，也支持拖拽粘贴的方式上传图片
- issue可以评论回复，与博客评论毫无区别，方便沟通
- git可以watch，甚至可以针对单条issue watch，极其有效的收到你的关注
- git action可以帮助自动化，相比hexo，流程要简单的多
- 无广告，全是内容，而且不用担心不维护会“倒闭”
综上，如果你像我一样“懒”，不妨试着先用git issue记录起自己的成长和进步。
## 搭建过程
### 下载或fork
链接：[https://github.com/yihong0618/gitblog](https://github.com/yihong0618/gitblog)
首先打开这个仓库，可以download，也可以直接fork
![image](https://user-images.githubusercontent.com/50072042/212846323-e917d5af-85dd-423d-ade8-d99327673bb3.png)
红框里的文件是我们需要用到的，其他都可以删了（backup放的是原博主的博客，速速删除）
### 新建自己的仓库 修改文件
新建仓库就不说了，如果是fork的只要把该删的删除了就行了。
将main.py和requirements.txt文件直接添加进去，添加generate_readme.yml文件的时候需要注意文件路径保持一致
![image](https://user-images.githubusercontent.com/50072042/212847399-f02876e5-73aa-4128-a46c-4c218389b722.png)
然后对这个文件做修改，需要修改几个地方
env中的name和email，需要换成自己的
![image](https://user-images.githubusercontent.com/50072042/212847482-ec06bb88-a4b5-4db3-9a9b-cacbd2d7358d.png)
branch换成对应的branch
![image](https://user-images.githubusercontent.com/50072042/212847588-c1749fa0-3bc5-47ac-8252-730205892d72.png)
### 配置token
点击settings，进入最下方有个developer settings
![image](https://user-images.githubusercontent.com/50072042/212847824-8227c0c7-1856-437b-a983-5fbc4087784f.png)
选择generate new token
![image](https://user-images.githubusercontent.com/50072042/212847912-4a8b0087-ccdd-447c-a6c4-4f130a5570ec.png)
这三个地方分别配置的描述，期限和权限
描述可以自己随意填写，另外两个可以按照自己的需求来
创建后，就会获得一个token了，复制一下（记得复制，你只能看见这个token一次）
然后来到我们刚刚创建的仓库，点击settings，选择new secret
![image](https://user-images.githubusercontent.com/50072042/212848248-c13c57ea-6e41-4722-b2a8-674090f4a1ca.png)
![image](https://user-images.githubusercontent.com/50072042/212848306-34ccb1a9-194b-4f76-bf58-69cd1a9b3b91.png)
这里的secret就是你刚刚复制的token，Name的话，之前复制的文件中用的G_T，所以你也可以就可以直接填G_T，如果用的别的话，可以去原先的generate_readme.yml文件中修改一下
![image](https://user-images.githubusercontent.com/50072042/212848390-e714a7e5-3437-49ee-becb-0f4ed01e4c9d.png)
### 开始使用
然后你就可以在issue中愉快的写东西了，当你发布一个issue后，readme会自动更新，还可以根据labels自动分类
### 一些问题
如果你发现你按照上述流程做了之后发现并没有什么用，可以去workflows中看看执行日志，查看到底是什么问题
![image](https://user-images.githubusercontent.com/50072042/212849035-2b304afb-6c89-49be-82cc-583a43fa78f3.png)
说一个我遇到的问题，在一开始的时候，我发现脚本执行了，但是没有push成功，查看日志后发现报错了permission denied，才发现是写权限没开
![image](https://user-images.githubusercontent.com/50072042/212849099-8d6f3396-cbe5-42bb-ae1b-a8bc81409737.png)
## 参考链接
脚本原作者：[https://github.com/yihong0618](https://github.com/yihong0618)
[https://github.com/yihong0618/gitblog/issues/177](https://github.com/yihong0618/gitblog/issues/177)
[https://zhuanlan.zhihu.com/p/400962805](https://zhuanlan.zhihu.com/p/400962805)
