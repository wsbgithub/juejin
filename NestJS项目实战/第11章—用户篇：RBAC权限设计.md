﻿## 前言

从本章开始是一道分水线，在这之前我们一起学习了 `NestJS` 的基础用法，通过搭建脚手架以及完成一些小需求逐步地熟悉了 `NestJS` 的开发模式。

首先，我们一起看一个非常熟悉的场景。张三在页面中点击了删除按钮后，系统在背后做了一些什么样的操作？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52142b047b764852a160e46821a5cf32~tplv-k3u1fbpfcp-watermark.image?)

一个用户在使用系统做某些操作的时候，系统会去数据库或者其他持久化的地方查询该用户所拥有的权限，然后根据查询出的结果判断此次操作是否正常。

简单的情况，一张表存储用户的权限，然后直接查询判断即可。但**用户量足够大又或者权限非常多**的话怎么办呢？一个新的系统需要接入用户、权限的时候，又该怎么办？

带着这些疑问，本章将介绍如何去设计一个**可拓展的**用户权限系统？

## RBAC 权限设计
### 什么是 RBAC 模型

为了解决前述的问题，我们将引入 **RBAC** 权限管理设计。

> **RBAC（Role-Based Access Control）** 的三要素即**用户**、**角色**与**权限**。 用户通过赋予的角色，执行角色所拥有的权限。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1683434857d14024bf5318a679af4666~tplv-k3u1fbpfcp-watermark.image?)

**RBAC** 引入之后的流程如上图所示，用户在进入系统之后，会先进行角色判断，再根据对应的角色查询所匹配的权限，最后根据返回结果来判断是否可执行。

直观来说，整个调用的链路被拉长了，直接使用用户与权限的绑定关系，明显速度会更快。

所以为什么不直接使用**用户 -> 权限**的链路而是采用**用户 -> 角色 -> 权限**的链路呢？

通过下述的表格数据，我们来对比一下两个方案的差别：

|  方案 | 用户量 | 权限数 | 权限表数据量 
| --- | --- | --- | --- |
| **用户 -> 权限** | **1** | **10** | **10 * 1** |
| **用户 -> 权限** | **100，000** | **10** | **10 * 100，000** |
| **用户 -> 角色 -> 权限** | **1** | **10** | **10 * Role** |
| **用户 -> 角色 -> 权限** | **100，000** | **10** | **10 * Role** |

上面的数据可能看得有些懵懂，我们转换文字版本来解释一下：

如果一个用户拥有 **10** 个权限的话，使用用户权限关联表后，一个用户就会有 **10** 条数据，**10** 万个用户的话就有 **100** 万的数据，代表着当一个用户进入系统之后，我们需要在**百万级别的数据表**中查询对应的权限数据。

而使用 **RBAC** 之后，当用户进入系统之后，先查询用户对应的角色，再查询角色映射对应的权限表，即便是一个角色对应一个用户，那么查询量也就是在 **10 * 10** ，比直接查询百万数据表的数据量直线下降，如上对比可以看出，使用 **RBAC** 能大量节约查询成本与时间。

同时一个角色可以挂载多个权限，从实际使用场景、覆盖的范围以及性能优化上都比单纯的**用户-权限**表更高效。

### RBAC 模型的分类
**RBAC** 模型可分为 **RBAC0**、**RBAC1**、**RBAC2**、**RBAC3**，其中 **RBAC0** 是基础模型。 **RBAC1**、**RBAC2**、**RBAC3** 都是在 **RBAC0** 模型的基础上升级。

#### RBAC0 模型

**RBAC0** 即最简单的用户角色权限管理模型：

- 用户和角色可以是一对多，一个用户只能赋予一个角色； 一个角色可以关联多个用户
- 用户和角色可以是多对多的关系， 一个用户拥有多个角色；一个角色可以关联多个用户

通常在功能简单，用户人员较少，并且用户岗位很明确，而且用户不会兼任时使用一对多关系；其余情况普遍采用多对多的关系。

基于此模型设计数据库表如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25a84dce643d4a688af1026671814efd~tplv-k3u1fbpfcp-watermark.image?)

#### RBAC1 模型

基于模型 **RBAC0** 的升级版本，一个角色可以从另一个角色继承许可权，即角色具有上下级的关系。

一个简单的例子，**GitLab** 中 **master** 与 **dev** 分为两种角色，**matser** 的权限会涵盖 **dev** 所有的权限，也就是 **master** 继承了 **dev** 的权限，同时额外增加了更高级别的权限。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2759d0ce19b04b1498cc8e3ed9e38126~tplv-k3u1fbpfcp-watermark.image?)

角色间的继承关系可分为一般继承关系和受限继承关系：
- 一般继承关系允许角色间的多继承，无特殊限制；
- 受限继承关系则进一步要求角色继承关系是一个树结构，也就是继承关系受到限制，继承 **A** 类后的角色不再允许继承同级角色 **B** 类，等同于测试与开发的子权限不能互相继承。

简单的表达关系如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5772933eb57d47068e6498e0d07a8f25~tplv-k3u1fbpfcp-watermark.image?)

#### RBAC2 模型

**RBAC2** 模型是在 **RBAC0** 模型基础上解决了角色的授权场景。角色授权分为两类：
- 静态职责分离    
- 动态职责分离 

静态职责分离又具体分为：
1. 角色互斥 -- 多种角色间不能同时赋予同一个用户，比如 **devops** 中，研发、产品与测试的权限不会相互重复赋予，当然你可以设置一个更高级别的角色权限 **leader** 来同时享用所有权限。
2. 基数约束 -- 角色至多能赋予 **N** 个用户。
3. 先决条件角色 -- 授予用户 **B** 角色前提是用户必须已经拥有 **A** 角色，这个在项目管理中比较常见，当你想给你组员分配角色时，你的角色权限理应高于需要分配的角色。

动态职责分离即运行时通过当前会话确定用户角色。例如以我们的范例飞书来说，在飞书账号中可以有多重公司认证，但登录的时候只能选择确定的一家公司身份才能进入。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a74a10e944e4fd2b21c010a8f6ab83b~tplv-k3u1fbpfcp-watermark.image?)

#### RBAC3 模型
**RBAC3** 模型是目前最全面的权限管理，它是基于 **RBAC0** 的基础上，并将 **RBAC1** 和 **RBAC2** 进行了整合。模型示例如下图所示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/947ee692cda447269d167c821a07b9e2~tplv-k3u1fbpfcp-watermark.image?)

## 进阶 - 用户组

当系统用户非常多以及角色种类非常多的情况，为了更方便的管理人员，此时可以引用用户组的概念。

每一个用户组分配一批用户，再将角色分配到用户组，将用户与角色之间的桥接关系再引入一层用户组，使得用户只与用户组绑定，用户组与角色绑定。当新的用户想要分配权限的时候，可以直接添加到对应的用户组，这样快速开通该用户组的所有权限，再根据需求分配更细节的角色即可。

这个场景的实例可以参考 **GitLab** 的 **Group** 管理模式。

- 用户可以拥有单独的角色权限
- 用户分配到用户组就可以自动拥有用户组的角色权限

这种设计大大减少了数据的冗余性和管理员对权限管理的工作复杂程度。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b7265490e764a49b3a2220e43494c73~tplv-k3u1fbpfcp-watermark.image?)

## 权限的拓展

当系统逐渐庞大后，权限也需要更加的粒度细化。对于权限的管理分为功能权限和数据权限：

- 功能权限：将系统的可操作性分配给角色，来控制用户的可见性和可编辑性
    1. 读写权限：可见可编辑，
    2. 只读权限：仅可见不可更改
    3. 不可见权限：不可见也没有操作入口
- 数据权限：数据是多维的、抽象的，主要控制某条数据记录对用户是否可见，结合功能权限可以更灵活的配置业务过程中每一位员工的功能操作权限及数据可见范围
    1. 基础数据：比如只有创建人可编辑，其他人只读
    2. 数据共享：比如部门 **A** 的所有成员均可查看部门 **A** 的全部处理的财务记录。

针对功能权限按功能类型可分为菜单模块、页面元素模块、文件资源模块等。要结合实际业务需要合理划分功能点来控制权限的粒度，将权限拆分到模块可以方便后续的其他类型模块拓展。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b4a90a89e93423187917875ee11c781~tplv-k3u1fbpfcp-watermark.image?)

针对于上述所有的拓展与设计，最终版本的设计如下所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/192cff00645e41759d06859323f589d4~tplv-k3u1fbpfcp-watermark.image?)

## 项目实战设计

真实的项目中，建议按照上述的模式开发，整体功能完整性与拓展性都会比较好，但是对于我们的系统而言，有点重量级，所以并不会完全按照上述的架构设计开发功能。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d24caa0187c544c5933aff6b50af9b8d~tplv-k3u1fbpfcp-watermark.image?)

在我们的需求设计中，用户系统需要分别针对两个系统提供鉴权服务，借助用户组的概念，最终用户系统的用户权限模型为下图所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ba2555d099b4011bcc9e98d009c024a~tplv-k3u1fbpfcp-watermark.image?)

> 下面是权限操作界面的网图，给大家做一个参考，实际我们的项目并没有前端的项目开发，只涉及后端开发，如果有需要或者有兴趣可以参考下图自行实现。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08cfaa8b8d8b4751a26889f65abf1a9a~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/022d90ce9b204af28b5713597b2c2816~tplv-k3u1fbpfcp-watermark.image?)

## 写在最后

用户系统的代码已完成初版，需要的同学可以自取 [feat/user](https://github.com/boty-design/gateway/tree/feat/user)，相关的注释也已经补充完毕，如果感觉哪里需要修改或者不明白的地方可以随时与我沟通。

从本章开始将进入正式的实战环境，**从第十章开始一直到微服务的章节将全部都是真实的项目设计**，我们将通过各种项目的设计与思路来解析每个项目开发。

跟之前说的一样，由于每个人的编码习惯与真实需求不一致，所以不会跟学习篇一样，将代码以及步骤完完整整的搬到教程里面来，只是提供具体的设计方案与思路，但最后提供以教程中的架构实现的项目，所以记得关注 <https://github.com/boty-design/gateway>，同时进群获取最新的进度。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b5e343d74b549249a6a4a7d426b6fbd~tplv-k3u1fbpfcp-watermark.image?)

如果你有什么疑问，欢迎在评论区提出或者加群沟通。 👏

