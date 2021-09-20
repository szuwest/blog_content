Title: Code Review方案
Date: 2018-01-10
Category: 技术
Tags: codereview 

# Code Review方案

## 定义
Code Review代码评审是指在软件开发过程中，通过对源代码进行系统性检查的过程。通常的目的是查找各种缺陷，包括代码缺陷、功能实现问题、编码合理性、性能优化等；保证软件总体质量和提高开发者自身水平

## code review 的好处

* 1. 提高代码质量。
* 2. 及早发现潜在缺陷，降低修改/弥补缺陷的成本。
* 3. 促进团队内部知识共享，提高团队整体水平。
* 4. 评审过程对于评审人员来说，也是一种思路重构的过程。帮助更多的人理解系统。
* 5. 是一个传递知识的手段，可以让其它并不熟悉代码的人知道作者的意图和想法，从而可以在以后轻松维护代码。
* 6. 鼓励程序员们相互学习对方的长处和优点。 
* 7. 可以被用来确认自己的设计和实现是一个清楚和简单的。

## code review形式
一般code review有两种形式，一种是代码评审会议，我称之为Code Review Meeting，就是将团队成员都组织起来开会，让代码Owner上去讲自己代码的实现和思路，其它人发表意见和进行讨论，也有把这种叫做team review。另外一种是一对一评审，我称之为Single Review，就是项目owner提交代码之后，让reviewer在空闲的时候帮忙评审代码，并且写出批注，owner收到批注后，进行修改或者回复。但注意这里的reviewer并不是只有技术主管或架构师之类的才能做，代码质量监管仅仅靠架构师是不够的，需要所有经验丰富或有专长的同学参与其中。也有人将这个形式叫peer review。
现在大部分公司都使用为Single Review形式，或者两者混合使用。

## code review 工具
我们这里只介绍single review形式的工具。现在有比较受好评code review工具有Facebook的Phabricator，Google的Gerrit，他们都是开源的.另外微软也有他的code review工具TFS(Team Foundation Server),据说也挺好用，不过是收费的。不过现在大家用得最多的code review方式是基于Pull Request工作流方法，结合gitlab或者github来使用。现在Git是最流行的代码管理工具，结合gitlab的pull request，很容易实现code review。

## Code Review流程
这里介绍一下基于gitflow+gitlab来做code review的流程。要在gitlab里做好code review需要有个前提，就是做好权限管理。每个成员在项目里都有对应的角色，例如owner，master，developer等。然后项目代码里设置受保护分支，master一定是受保护的分支，还可以根据需要设置其他分支为受保护分支。developer权限的成员是不能向master或者其他受保护分支push代码的。

**所以结合code review，开发中的整个流程就是：建立feature分支-->编写代码-->push分支代码-->gitlab上发起一个合并请求（pull request）-->审核人员审核代码，如有需要，提出修改意见-->开发人员修改代码-->审核人员审核通过，合并代码，删除分支**


下面介绍一下详细的流程，和对应的git操作命令：

1、根据开发任务，建立git分支, 分支名称模式为feature/任务名，比如关于API相关的一项任务，建立分支feature/api。
git checkout -b feature/api

2、运行git branch 确认切换到了feature/api分支

3、编辑代码完成开发任务， commit相关代码
git add -A
git commit -m "implement api architecture"

4、将分支代码push到服务器
git push origin -u feature/api

5、登录到gitlab源代码库，如http://192.168.0.2/native/record-app ，点击合并请求（Pull request）按钮去创建一个合并请求（pull request）

6、再pull request详细页面， 填写相关标题／说明／reviewer， 目前请将reviewer设成相关人员

7、请提醒reviewer去审核pull request，系统也会发邮件提醒reviewer

8、Reviewer打开pull request页面，查看代码修改情况，也可以在相应的代码处添加注视，提示代码作者哪里应该修正。

9、代码作者根据reviewer的要求，调整代码后commit／push到服务器。 然后reviewer继续设置， 如此循环，知道没有问题。

10、当代码没有问题以后， 需要将任务代码merge到主代码库， 有两种方法：
 a、Reviewer可以在pull request页面点击Merge按钮， 把代码merge到主代码库
 b、Reviewer手动本地merge， 并push到服务器。
git pull origin develop
git log ..develop

如果看到develop里有修改没在当前分支， 那么运行git rebase develop来把develop的修改加入到当前分支
运行一下合并命令
git checkout develop
git merge --no-ff feature/api
git push

11、代码作者删除feature子分支。
git checkout develop
git branch -D feature/api
git push origin :feature/api

git pull origin develop 


总结：核心流程就是 建立分支--发起PR请求--审核--合并，不断的循环反复。


## code review注意事项
待补充


## 参考资料
[学习笔记_Git之CodeReview流程](http://blog.csdn.net/june_y/article/details/50817993)

[使用gitlab做git flow及代码审查](http://blog.csdn.net/wh_19910525/article/details/68068397)

[Git工作流指南：Pull Request工作流](http://blog.jobbole.com/76854/)

[如何做好代码审查？Code Review Meeting还是Single Review
](http://blog.csdn.net/jsjwk/article/details/50379836)

[我们是怎么做Code Review的](https://www.cnblogs.com/wenhx/p/How-We-Code-Review.html)


[如何进行高效迅速的CodeReview](http://blog.csdn.net/huver2007/article/details/75095303)

[如何有效的做Code Review](http://blog.csdn.net/lackin/article/details/7754967)


知乎上的讨论：

[有人实践过 Phabricator 以及 Arcanist 作为 code review 的工具么？](https://www.zhihu.com/question/19977889)

[大家的公司的code review都是怎么做的？遇到过什么问题么？](https://www.zhihu.com/question/41089988)