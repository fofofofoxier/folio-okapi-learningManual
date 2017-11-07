# Okapi 安全模型（Okapi Security Model）

+ 介绍
+ 总览
+ 认证(Authentication)
+ 授权(Authorization)
+ 数据流示例(Examples of data flow)
+ + 简单的请求: 日期
  + 更复杂一点的请求：MOTD
  + 登陆
+ 公开的问题和技术细节(Open questions and technical details)
+ Okapi执行过程(Okapi process)
+ 认证授权过程(The auth process)

-------

## 介绍

Okapi的安全机制包含两个部分：认证(Authentication)和授权(Authorization)，认证即确定用户的真实身份，授权则是检验用户是否被允许做某个操作。跟授权相关，我们有很多 **原子权限**(permission bits，译者注：目前能够想到对"permission bits"最好的解释就是"原子权限"，即不可变的/或最小级别 权限)，我们还有一个能管理这些原子权限的系统。Okapi实际上不关心他们(原子权限)是否在一个独立的模块中管理还是在认证模块中的一部分管理。权限(permissions)有两个来源: 用户的权限以及模块的权限。

Okapi仅仅把这些权限视作普通的字符串(simple string)，就像"patron.read"、"patron.read.sensitive"，这些字符串对于Okapi来说没有任何意义，但是我们必须要开发出一些命名规范(naming guidelines)。一方面，模块本身应该定义出特定粒度的权限，以便于权限的控制。但是，另一方面来说，他们(permissions)的管理应该尽量简化。权限管理功能可以通过permission module来创建permission group来实现(译者注：对原子权限的组合，称为permissionSet或permission group, 也就是一般业务系统中的“角色”的概念，同时permissionSet中可以包含其他permissionSet)。例如，一个“系统管理员”角色，原文称sysadmin role，可能会扩展为一系列权限，包含"patron.admin"，同时"patron.admin"权限可以扩展成包含 "patron.read" "patron.update" "patron.create"等权限。这些全部都由permission module去管理。

Okapi的项目源码中包含了一个简单的module `okapi-test-auth`，这个module非常简略地实现了认证和授权这两个功能，它是用来测试自身功能的module。它不是一个真正的认证模块。实际上，认证和授权是在不同的模块中分别实现的。

## 总览

略过所有复杂的细节，如下过程能够大概描述一个用户使用Okapi时的流程：

+ 首先用户通过浏览器访问登录界面
+ 用户输入密码(credentials)
+ 授权模块校验密码
+ 授权模块生成一个令牌(token)，并返回给浏览器端用户
+ 用户在访问Okapi时的所有请求都带着这个token
+ Okapi访问认证模块并校验这是否是一个合法的用户，并校验该用户是否有权限进行访问
+ 模块本身可以通过Okapi对其他模块进行调用，模块再一次传递这个token，并且Okapi将这个token传递到授权模块进行校验。如果一个模块含有一些特殊权限，那么Okapi偶尔会传递一个升级版的(处理过的)token。

## 认证(Authentication)

很明显，我们需要不同的认证机制，例如SAML,OAuth,LDAP等，从而管理用户名和密码，以及IP认证，甚至需要对一些通用的公共用户做伪认证。

每一个tenant都应该至少被授予有一个认证模块。用户的首先应该请求 `/authn/login`接口，这个接口定向到了认证模块之中。认证模块会得到一些参数，例如 tenant, username, password。它会通过跟后台其他模块交互并校验这些数据。当校验成功之后，系统会通过授权模块生成一个JWT token，授权模块才是真正关心这些token的模块。无论如何，客户端会收到一个token。这个token会携带着userId和该用户所属的tenant，同时也可能携带其他信息。这个token是经过加密的，所以token的内容在没检测通过的情况下是不可以被修改的(此处翻译有可能不准确，附上原文：The token will always contain the userId, and the tenant where the user belongs, but may contain many other things too. It is cryptographically signed, so its contents can not be modified without detection.)。

模块本身可能也会从后端收到permission的数据，并且向授权模块(或者permission module)提交一个新的请求。

客户端在每次请求的时候都会带着系统生成的JWT，并且授权模块每次都要校验。

## 授权(Authorization)

这里更加与Okapi相关，并且事情变得更加有技术性。

当Okapi接受到一个请求(除了`/authn/login`这种对所有人开放的请求)，Okapi检查tenant，并确定一个modules的调用顺序`pipeline of modules`。其中授权模块在最开始的位置，或者说接近开始的位置。

同时，Okapi从ModuleDescriptors中检查所有模块里包含的permissions(原子权限)。总共有三种permissions:

+ `permissionRequied` ：访问module的必须权限。
+ `permissionDesired` ：非必需权限，但是，如果拥有这种权限，module可能会提供额外的操作(例如，显示一个patron的敏感信息)
+ `modulePermissions` ：跟前两种permission不同，这种权限是授予module的，module可以利用这个权限访问其它module的功能

正如之前提到，Okapi不对permission进行任何校验，它仅仅收集`permissionRequired`和`permissionDesired`两种权限，并将他们传入授权模块进行检查。Okapi也收集`pipeline of modules`中每个module的`modulePermissions`，并传入授权模块。

授权模块首先校验我们是否有一个合法的token，检查签名和内容是否匹配。然后从中读取user和tenant ID，并查找该user名下的所有permissions。同时也校验token中是否含有`modulePermissions`，如果存在，则将这些permissions添加进该user的permission list。接下来，校验所有`permissionRequired` 是否全部在user的 permission list中出现，如果有任何缺失，整个请求宣告失败。 授权模块也检查 `permissionDesired` ，然后检查他们在user permission list 中是否存在，这个信息在一个特殊的`header`中，这样接下来的modules便可以看见他们，并决定更改本次请求的行为。

最终，授权模块查看它接受的`modulePermissions`。对于每一个 module pipeline中的 module, 授权模块会创建一个新的JWT，这个JWT包含原来信息以及模块附加的permission信息。授权模块生成这个JWT，并将它存到`X-Okapi-Module-Tokens`的header里，并全部返回到Okapi。如果原来的token已经包含了一些module permissions，那么授权模块也会生成一个新的token，这个新的token不包含原本token中已经存在的module permissions，这用于那些没有特殊权限的modules。这将或多或少地与用户的原始令牌相同。

Okapi检查请求中是否含有module tokens，并存储到目前所执行的 Module pipeline 当中，这样以便于向有特殊权限的module传递特殊的token。其他modules则是一个clean token。

接下来所要执行的modules不知道也不关心他们得到的token是否是从UI界面获得的原始token，还是特地为他们生成的token。他们仅仅将token传递跟下一个要访问的module，Okapi会负责校验。

如果一个请求没有任何JWT，那么这是一个特殊情况。例如，用户希望登陆系统。授权模块会为Okapi生成一个临时的token以供使用。这个JWT很显然不会包括username，因此这次请求也不会被授予有任何的用户权限。但是，这个token可以用来访问那种不要求任何权限的module，并且可以用作module之间传递特殊权限的基础。这个JWT会在Okapi发起请求之后被Okapi抛弃。

这会是一个安全风险吗？No it is not : 这个临时的JWT不提供任何的permissions。调用者也许会在发出请求时省略这个JWT，但这是无用的。一种情况是：请求不需要任何权限，在这种情况下，无需JWT就可以调用它；另一种情况：请求确实需要某种权限，因为大多数情况都很可能这样做，在这种情况下，请求将被拒绝，因为临时JWT没有提供任何权限。