## CoolPlayFlink



## Pull Request流程

开始对Pull Request流程不熟悉，后来参考了[@numbbbbb](https://github.com/numbbbbb)的《The Swift Programming Language》协作流程，在此感谢。

1. 首先fork该项目
2. 把fork过去的项目也就是你的项目clone到你的本地
3. 运行 `git remote add coolplaydata git@github.com:coolplaydata/coolplayflink.git` 添加为远端库
4. 运行 `git pull coolplaydata master` 拉取并合并到本地
5. 翻译内容
6. commit后push到自己的库（`git push origin master`）
7. 登录Github在你首页可以看到一个 `pull request` 按钮，点击它，填写一些说明信息，然后提交即可。

1-3是初始化操作，执行一次即可。在翻译前必须执行第4步同步我的库（这样避免冲突），然后执行5~7既可。
