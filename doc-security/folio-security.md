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



## 数据流示例

以下是一些实际的例子，这些例子解释了系统不同部分之间的数据流传输的方式。我们从最简单的开始，循序渐进，最后一个例子是最复杂的：登陆流程(logging in)。

### 简单的请求：日期(Date)

**假设这样的情景**：我们的UI层需要显示当前日期，我们的用户已经登陆进了系统，所以 已知userId("joe") 和 tenantId("ourlib")。我们还有一个供授权使用的 JWT，至于JWT的内容细节是什么 这无所谓(Details are not relevant)，在这些例子当中 我们假设 JWT 长成这个样子："xx-jow-ourlib-xx"。UI层知道okapi-server服务的地址，我们用`http://folio.org/okapi`作为示例当中的base-uri。我们还有一个提供日期服务的module("cal")，这个module已经被授权于 `ourlib` 这个租客(tenant)，并且 该module的一个服务地址为 `/date`, 这个服务地址返回值是当前的日期，这个服务对所有人开放。

###### 概览：

+ 1.1: UI发出请求
+ 1.3: Okapi调用auth module，该module校验JWT
+ 1.5: Okapi调用cal module从而获取当前日期

###### 细节：

**1.1: UI携带一些额外的header去请求Okapi**

+ Get `http://folio.org/okapi/date`
+ X-Okapi-Tenant: ourlib
+ X-Okapi-Token: xx-joe-ourlib-xx

**1.2: Okapi接受请求**

+ 检查tenant值为"ourlib"
+ 为"ourlib"所启用的所有`/date`服务相关的module被构建成为了一个List。
+ 这个List显然存有两个Modules: 首先"auth"，然后是实际提供服务的module "cal"。
+ Okapi 发现请求头中包含着 X-Okapi-Token，然后将JWT存储起来以供将来使用。
+ Okapi检查 required-permission和desired-permission，本例中都不存在，Okapi创建了另外两个Header：X-Okapi-Permission-Required 和 X-Okapi-Permission-Desired，这两者都是empty list。
+ Okapi检查module是否被授予了permission，事实上本例并没有。所有Okapi创建了一个没有内容的header：X-Okapi-Module-Permissions 
+ Okapi检查 auth module运行在什么位置(哪个节点，哪个端口)
+ Okapi 将请求转发到auth module，此时请求中多出了如下的headers：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx
  + X-Okapi-Permissions-Required: [ ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { }

**1.3: auth-module接受请求**

+ auth-module检查到请求中存在一个 X-Okapi-Token的header。
+ 校验该token的签名。
+ 从中提取tenantId和userId
+ 校验tenantId是否和X-Okapi-Tenant header中的值相符合
+ 发现没有任何required permissions，没有module permissions，所以构建一个空的X-Okapi-Permissions header
+ 发现没有包含特殊的moduel-permissions，所以创建一个空的X-Okapi-Module-Tokens header
+ auth-module向Okapi发送一个OK的response，该response携带以下几个新的headers：
  + X-Okapi-Permissions: [ ]
  + X-Okapi-Module-Tokens: { }

如果以上有任何一步执行失败了，auth-module会返回一个error-response，Okapi随后会停止执行module-pipeline里面的任务，并向UI返回一个error-response。

**1.4: Okapi从Auth-module接受到OK-resposne**

+ okapi接受到一个X-Okapi-Module-Tokens header，这表示授权操作已经完成，所以他可以将 X-Okapi-Permission-Required 和 X-Okapi-Permission-Desired headers从接下来的请求中移除掉。
+ okapi发现header中没有Token，所以不做什么额外的事情
+ okapi看到下一个需要调用的module是"cal module"
+ 检查cal-module在哪里运行(地址，端口号)
+ 请求cal-module，并携带如下的headers：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx
  + X-Okapi-Permissions: [ ]

**1.5: cal-module接收到请求:**

+ cal-module查询当前日期并用合适的方式格式化显示日期
+ 返回给Okapi一个OK-response，不添加特殊的headers

**1.6: Okapi 接收到OK-response**

+ okapi发现这是list中的最后一个module
+ 将返回值响应给UI

**1.7: UI在屏幕上显示日期**

整个流程也许过于复杂，但是下一个例子将会展示需要如此繁杂步骤的原因。

注意一件事：一个request也许不是必须要发起自UI层面，任何能够初始化一个request的module都可以发起一次request，这些请求可能发起于一个在中午或者晚上执行的定时任务，也可能发起于一个 book-returning machine(译者注:此句原文为 **"It can come from anything that can initiate a request, a cron-like job that wakes up in the middle of the night, or a book-returning machine that saw a returned item"** ,此句如果直译 则过于苍白，容易给人以误导，所以放出原文，体会类比)。然而无论怎么样，他们必须有一个JWT。



## 更复杂一点的请求:MOTD

The Message of the Day(MOTD) 是一个更加复杂一点的请求示例，在这个例子中，我们有两种请求，一种是顾客(patrons)发起的请求，另一种是工作人员请求(staff only)。staff members可以看见staff message，但是patrons看不见。一般的patrons 值可以看见patron message。公共不明身份的用户(Unidentified members of general public)不允许访问任何资源。

message存储与database。motd-module需要向database-module发出请求，databse-module需要接收这个请求。为了做到这一点，motd-module需要一个读取database-module的权限。

###### 概览：

+ 2.1: UI发起request
+ 2.2: auth-module校验JWT，查找user的权限
+ 2.7: 获取permission的请求通过okapi，然后转发到perm-module(该module可以获取用户的权限)
+ 2.9: auth-module校验用户权限
+ 2.9: auth-module为motd-module创建一个特殊的JWT，里面携带着permissions信息。
+ 2.11: motd-module向db-module发起请求(携带着特殊JWT)
+ 2.12: auth-module校验这个JWT，查看并检验权限
+ 2.15: db-module查询message
+ 2.16: message被返回到UI以供显示


###### 细节：

**2.1: UI向Okapi发起request，携带如下headers:**

+ GET `http://folio.org/okapi/motd`
+ X-Okapi-Tenant: ourlib
+ X-Okapi-Token: xx-joe-ourlib-xx

**2.2: Okapi接收到请求**

+ okapi查到tenant值"ourlib"
+ okapi构建一个授予"ourlib"的并且服务于`/motd`的module-list。
+ module-list存在两个modules，首先是auth-module，然后是实际提供服务的motd-module
+ okapi发现request中包含X-Okapi-Token ,像之前一样，存起来以供将来使用。
+ Okapi查看这些module的moduleDescriptors，检查required-permission和desired-permission。访问auth-module不需要任何权限。motd-moudle 必要权限(required)是 "motd.show"，非必须权限(desired)是 "motd.staff"，okapi把这些值放进X-Okapi-Permissions-Required 和 -Desired headers。
+ okapi检查moduleDescriptors 看看是否有权限被授予这些modules。motd-module有 "db.motd.read"权限。okapi把这个权限放入X-Okapi-Module-Permissions header。
+ Okapi检查auth-module在哪里运行(节点，端口)
+ okapi向anth-module转发请求(带有如下headers):
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx
  + X-Okapi-Permissions-Required: [ "motd.show" ]
  + X-Okapi-Permissions-Desired: [ "motd.staff" ]
  + X-Okapi-Module-Permissions: { "motd": "db.motd.read" }

**2.3: auth-module接收到请求：**

+ auth-module检查X-Okapi-Token header
+ 校验token签名
+ 提取tenantId和userId
+ 校验此tenantId是否和X-Okapi-Tenant相符合
+ 检查required/desired permissions。因为目前还没有任何 Joe 的权限被缓存，auth-module需要从permission-module中获取joe的权限。
+ auth-module发起了一个 `/permissions/joe`的请求，幸运的是，对permission-module的读操作访问不需要任何特殊的权限，至少在用户查询自己的权限时不需要这个限制。
+ auth-module向okapi发送请求：
  + GET `http://folio.org/okapi/permissions/joe`
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx

**2.4: Okapi 接收到请求，类似于之前1.2描述的那样：**

+ 检查"ourlib"tenant
+ 为"ourlib" 构建`/permissions/:id`服务的module-list
+ module-list包含"auth-module"和 "perm-module"
+ 查看X-Okapi-Token header，存储，备用
+ 检查module-list中所有modules的required/desired permissions。permission-module的查询功能不做权限限制这一点至关重要，因为我们要避免因为权限而引起的无尽的递归调用。
+ permission-module本身不能够拥有模块级权限是不需要理由的。例如："db.permissions.read"。现在我们假设他不存在。
+ okapi检查perm-module在哪里运行(节点 端口)
+ okapi将请求转发到auth-module：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx
  + X-Okapi-Permissions-Required: [ ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { }

**2.5: auth-module校验JWT，类似1.3:**

+ 返回给okapi一个OK response：
  + X-Okapi-Permissions: [ ]
  + X-Okapi-Module-Tokens: { }

**2.6: Okapi接收到OK response(没有任何额外的pwemission信息，类似于1.4)：**

+ 发现下一个应该调用perm-module
+ 检查perm-module在哪里运行
+ 向perm-module发起请求：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx
  + X-Okapi-Permissions: [ ]

**2.7: perm-module接收请求:**

+ 找到joe的权限
+ 返回permission list 类似于[ "motd.show", "motd.staff", "what.ever.else" ]

**2.8: Okapi收到OK reponse：**

+ module-list中没有剩余
+ 向auth-module返回响应

**2.9: auth-module获取从perm-module的response,随后继续：**

+ X-Okapi-Permissions-Required中 存在"motd.show"
+ 查看JWT中没有特殊的module-permissions ，所以用了joe的permission-list
+ 缓存joe的permission-list以供后续使用
+ 检查joe的permission-list是否存在required-permission，如果没有，则立即返回error-response，okapi直接返回给UI错误信息
+ 发现X-Okapi-Permissions-Desired中 存在"motd.staff"，然后在X-Okapi-Permissions header添加 ["motd.staff"]
+ 接下来auth-module为motd-module查看X-Okapi-Module-Permissions
+ 所以 新建一个JWT，内容包含旧的JWT，外加上一个新的属性"modulePermissions",其值为"motd"，这个新的token看起来长这个样子：xx-joe-ourlib-motd-xx
+ 将新的JWT放置进入X-Okapi-Module-Tokens header
+ 最终返回OK response：
  + X-Okapi-Permissions: [ "motd.staff" ]
  + X-Okapi-Module-Tokens: { "motd" : "xx-joe-ourlib-motd-xx" }

注意：这个模块没有必要返回 "motd.show" permission，因为这个权限被严格的要求必须存在，如果joe没有required-permissions，那么auth-module就会执行失败，motd-module也就永远都不会被调用。

**2.10: Okapi从auth-module获得ok-response:**

+ okapi接收到了 X-Okapi-Module-Tokens header，这说明授权工作已经完成，所以可以把X-Okapi-Permissions-Required和-Desired headers从request中移除
+ 发现有一个"motd" 的 module-token。okapi将token存储到将要访问的module-list当中的motd-module当中
+ 下一个要访问的module是motd-module
+ 检查motd-module在哪里运行
+ 发现一个以供访问motd模块的 module-token
+ 想motd-module发起请求：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-motd-xx
  + X-Okapi-Permissions: [ "motd.staff" ]

**2.11: motd-module 接收到请求:**

+ motd-module 发现 X-Okapi-Permissions 中包含"motd.staff"权限，所以 决定从database中获取staff-only的信息
+ 向 `http://folio.org/okapi/db/motd/staff`发起请求，携带如下headers：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-motd-xx

**2.12: Okapi接收请求，然后像前述过程一样执行流程：**

+ 检查 tenant为"ourlib"
+ 为ourlib构建`/db/motd`服务的module-list
+ module-list包含“auth-module”和“db-module”
+ Okapi发现存在X-Okapi-Token，存储备用。
+ Okapi检查module-list中所有模块的required/desired permissions。db-module要求具有 "db.motd.read"权限
+ 为了简化流程，我们假设db-module本身没有任何特殊权限的限制
+ Okapi检查auth-module在哪里运行(节点,端口)
+ Okapi将请求转发到auth-module，携带如下headers:
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-motd-xx
  + X-Okapi-Permissions-Required: [ "db.motd.read" ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { }

**2.13: auth-module接收请求:**

+ auth-module检查X-Okapi-Tenant header
+ 校验token签名
+ 从token中提取tenantId和userId
+ 校验该tenantId于X-Okapi-Tenant header中的tenant值是否相符合
+ auth-module查看required/desired permissions，现在我们在缓存当中已经有了joe的permissions，所以auth-module用了这个permission-list
+ auth-module发现JWT当中有了特殊的modulePermission，于是在permission-list当中添加了"db.motd.read"权限
+ 它(auth-module)发现X-Okapi-Permissions-Required header中有一个"db.motd.read"权限，检查permission-list中的权限进行对比，发现这个权限存在于permission-list当中
+ 此时没有任何desired permissions
+ db-module也没有任何的module permissions
+ 因为 JWT 当中含有特殊的 module permissions，所以auth-module需要创建一个新的 JWT(没有module permission信息的新 JWT)，目的在于供之后再次调用其他module时使用。auth-module将module permission从JWT当中剔除出去，并进行记录。结果和最原始的哪个JWT是相同的，"xx-joe-ourlib-xx"。auth-module随后将它返回，返回值就存在于X-Okapi-Module-Tokens header当中，以一个特殊的module `"_"` 当中进行存储。
+ 返回一个OK-response到Okapi，携带如下headers：
  + X-Okapi-Permissions: [ ]
  + X-Okapi-Module-Tokens: { "_" : "xx-joe-ourlib-xx" }

**2.14: Okapi 从auth-module接收到OK-response**

+ Okapi发现一个`"_"`的特殊module permission。随后调用任何一个module的时候，这个token都会被复制，携带于header中，同时也覆盖了具有motd的特殊权限的token。通过这种方式，这些permission不会泄露到其他的module当中。
+ Okapi发现 不存在module tokens
+ 发现下一个应该去调用的是db-module
+ 向db-module发起一个request，携带如下的headers:
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-joe-ourlib-xx

**2.15: db-module接收到了request:**

+ 为staff用户查询`The Message of the Day(motd)`
+ 返回OK-response

**2.16: Okapi获取OK-response:**

+ Module list当中没有其他的modules，Okapi返回一个OK-response

**2.17: motd-module 从db-module收到了OK-response:**

+ 构建一个OK-response，响应消息中存储着db中的消息
+ 返回响应消息到Okapi

**2.18: Okapi从db-module获取OK-response**

+ 因为motd-module是module-list当中的最后一个，okapi将相应信息返回到调用者

**2.19: UI显示`Message of the day(motd)`**



## 登陆

以下是一个例子，这个例子描述里用户登陆到系统中事件流程纵览。在这之后我们再介绍繁杂的细节。

+ UI向Okapi发出请求，此时还没有任何JWT
+ Okapi转发请求到auth-module
+ auth-module发现请求中没有任何JWT，所以它创建了一个JWT
+ 它也会为authentication-module创建一个JWT，这个JWT是带有模块级权限(module-permission)的
+ Okapi携带着这个JWT(这个JWT让本次请求有了发起其他请求的权限)  调用authentication-module
+ authentication-module调用authorization-module为用户生成一个真正的JWT
+ 这个JWT会返回到UI层，以供将来后续的调用而使用

在这个例子当中，通过用户名密码方式登陆，但是其他的module可以检查LDAP服务器、OAuth或任何其他身份验证方法。

**3.1: UI启动，向用户展示登录页面：**

+ 用户输入认证信息，在本例中使用的是用户名 密码
+ 点击登录按钮

**3.2: UI向Okapi发起请求:**

+ POST `http://folio.org/okapi/authn/login`
+ X-Okapi-Tenant: ourlib
+ 当然，现在我们还没有JWT

应该引起注意的是: URL和tenantId在UI层必须已知

**3.3: Okapi收到请求:**

+ Okapi检查到tenantId为"ourlib"
+ 构建一个 服务于"ourlib"的`/authn/login`服务 的module-list
+ module-list中包含两个modules，首先是auth-module 其次是login-module
+ Okapi检查到X-Okapi-Token没有值
+ Okapi检查moduleDescriptors，寻找到这些module所需要的各种权限(required/desired permissions)，很明显 在现在这种情况下我们不能要求任何的权限，login服务必须是开放的
+ Okapi检查ModuleDescriptors，旨在查看这些module中是否被授予了模块级权限。login-module至少拥有"db.user.read.passwd" "auth.newtoken"两个权限，也许会更多。Okapi把这些权限信息放入X-Okapi-Moduel-Permissions当中
+ Okapi检查到auth-module在哪里运行(节点，端口)
+ Okapi转发请求到auth-module，携带如下请求头：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Permissions-Required: [ ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { "login": [ "auth.newtoken", "db.user.read.passwd" ] }

**3.4: auth-module接收到请求:**

+ 它(auth-module)发现请求当中没有任何的X-Okapi-Token
+ 所以新建一个JWT，这个JWT携带着从X-Okapi-Tenant获取的tenantId，但是没有userId。我们不妨将这个JWT描述为"xx-unknown-ourlib-login-xx"。auth-module将这个token以通用token返回，以`"_"`module为标示
+ 它发现我们不要求任何permissions
+ 它发现我们拥有模块级权限(module permissions)
+ 于是它为login-module在X-Okapi-Module-Permissions中列出的权限 创建一个新的JWT，"xx-unknown-ourlib-login-xx"，其中携带着 "auth.newtoken" 和 "db.user.read.passwd"权限
+ 然后它向Okapi返回一个OK-response，并携带如下的请求头：
  + X-Okapi-Permissions: [ ]
  + X-Okapi-Module-Tokens: { "_" : "xx-unknown-ourlib-xx", "login" : "xx-unknown-ourlib-login-xx" }

**3.5: Okapi接收到Ok-response:**

+ Okapi 发现我们拥有了module-tokens，于是将他们复制到将要被调用的module-list中。login-module将要获得"xx-unknown-ourlib-login-xx"，其他所有的module(这里auth-module已经被调用过了)会获取"xx-unknown-ourlib-xx"
+ 发现下一个需要调用的是login-module
+ 向login-module发送一个请求，携带请求头：
  + the request body that it received
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-login-xx

**3.6: login-module接收到信息:**

+ login-module发现请求中带有username 和password，它需要从db中找到相应的记录
+ 建立一个请求：
  + GET `http://folio.org/okapi/db/users/joe/passwd`
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-login-xx

**3.7: Okapi接收到请求，类似于2.12对其进行处理:**

+ 检查出tenant值为"ourlib"
+ 为"ourlib"的`/db/users`服务建立一个module-list
+ module-list包含"auth-module" 和 "db-module"两个
+ Okapi发现请求中带有X-Okapi-Token，将其保存，供以后使用
+ Okapi检查module-list中所有服务的所有要求的权限(required/desired permissions)，其中db-module要求"db.user.read.passwd"
+ 简而言之，我们假设db-module本身不要求任何权限
+ Okapi将请求转发给db-module，携带下列请求头:
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-login-xx
  + X-Okapi-Permissions-Required: [ "db.motd.read" ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { }

**3.8: auth-module接收到请求:**

+ 它(auth-module)检查到X-Okapi-Token的存在
+ 校验token签名
+ 从token中提取tenantId，此时没有userId
+ 检查tenantId是否和X-Okapi-Tenant相符合
+ 它发现我们要求必须拥有一些权限。因为我们在JWT中没有任何user的信息，所以不能查询任何权限
+ 它发现JWT当中含有modulePermissions，所以在本次请求中添加上"db.user.read.passwd"和 "auth.newtoken"权限 到目前还为空的permission-list
+ 它发现我们有一个X-Okapi-Permissions-Required的header，其中有存在一个值"db.user.read.passwd"。然后检查permission-list，发现permission-list当中存在header里面的这个权限值
+ 没有发现desired permissions
+ 没有发现任何db-module的模块级权限(module-permission)
+ 因为JWT中存在特殊的module-permissions，auth-module需要创建一个没有module-permissions的新JWT，以供以后调用其他模块使用。它将module-permissions剔除出JWT并记录下来。剔除后的JWT和最原始的是一样的，"xx-unknown-ourlib-xx"。它在X-Okapi-Module-Tokens请求头中，把这个token以特殊模块`"_"`作为标记
+ 返回OK-response给Okapi，携带如下headers:
  + X-Okapi-Permissions:[]
  + X-Okapi-Module-Tokens: { "_" : "xx-unknown-ourlib-xx" }

**3.9: Okapi从auth-module接收到了OK-response**

+ Okapi发现特殊module`"_"`的module token，随后将它复制到所有要调用的module中，覆盖了那个具有login功能权限的token
+ 发现没有其他的module tokens
+ 发现下一个要调用的是db-module
+ 向db-module发起请求，并携带如下请求头：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-xx

**3.10: db-module从Okapi接收到请求：**

+ 获取joe用户的密码散列,`hash for the password`(译者注：原文是 ‘ It looks up the hash for the password for "joe".’ 请参照原文理解)
+ 返回OK-response

**3.11: Okapi收到Ok-response:**

+ module-list中没有了其他module，okapi返回一个Ok-response到login-module

**3.12: login module 收到了携带着`hash for the password` 的响应:**

+ login-module校验用户输入的密码散列与db查询到的散列进行对比，看是否符合
+ 此时我们为用户进行了认证
+ 接下来 login-module需要为这个用户新建一个JWT。这件事不是由login-module自己完成的，它去调用auth-module去做这件事，它向auth-module发起了一个请求，携带如下headers:
  + POST `http://folio.org/okapi/auth/newtoken`
  + username in the payload
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-login-xx

**3.13: Okapi 接收到了请求，执行类似3.7的流程：**

+ Okapi检查到tenantId为"ourlib"
+ 为"ourlib"的`/auth/newtoken`服务构建一个可供调用的module-list
+ module-list包含两个module，首先"auth-module",随后还是"auth-module"，但是后者访问的服务路径是`/newtoken`
+ Okapi发现请求中包含X-Okapi-Token，将token存储，以供将来使用
+ Okapi检查module-list中所有modules要求的权限，auth-module的`/newtoken`服务要有必须有"auth.newtoken"
+ Auth-module本身不要求任何特殊权限的限制
+ Okapi将请求转发到auth-module，携带如下headers:
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-login-xx
  + X-Okapi-Permissions-Required: [ "auth.newtoken" ]
  + X-Okapi-Permissions-Desired: [ ]
  + X-Okapi-Module-Permissions: { }

**3.14:auth-module接收到请求:**

+ 发现X-Okapi-Token
+ 校验token签名
+ 从中提取tenantId，此处依旧没有userId
+ 校验tenantId和X-Okapi-Tenant中的tenantId是否符合
+ 它发现我们要求必须有一些权限。因为我们在JWT当中没有user信息，所以不能查询任何permissions
+ 它发现JWT中有模块级权限，所以将"db.user.read.passwd"和"auth.newtoken"添加到permission-list当中
+ 它看见我们的X-Okapi-Permissions-Required中 有"auth.newtoken"，随后到permission-list当中寻找这个值，发现这个值在permission-list当中存在
+ 没有发现任何desired permissions
+ 没有发现任何db-module的模块级权限
+ 因为JWT中已经含有特殊的模块级权限，auth-module需要创建一个不存在这些特殊权限的新JWT，以供将来调用其他模块而使用。将modulePermission剔除出JWT并记录下来，剔除后的结果和原始JWT相同，即"xx-unknown-ourlib-xx"。随后将这个token以一个特殊module `"_"`为标示，放入X-Okapi-Module-Tokens中返回。
+ 返回给Okapi一个OK-response，并携带headers:
  + X-Okapi-Permissions: [ ]
  + X-Okapi-Module-Tokens: { "_" : "xx-unknown-ourlib-xx" }

**3.15: Okapi从auth-module接收到响应：**

+ 发现特殊的module token ：`"_"`。在随后需要被访问的module中，这个token都会被复制，携带，覆盖了具有login功能的token
+ 发现没有其他的module tokens
+ 发现下一个要访问的module依旧是auth-module，这一次 服务路径为`/newtoken`
+ 向auth-module发送请求,携带如下headers：
  + X-Okapi-Tenant: ourlib
  + X-Okapi-Token: xx-unknown-ourlib-xx
  + the original payload with the username

**3.16: auth-module接收到请求：**

+ 生成一个JWT，这个JWT具有从请求中获取的username信息 和 从X-Okapi-Tenant获取的tenant信息。不妨认为这个JWT长成这个样子：`"xx-joe-ourlib-xx"`
+ 返回OK-response

**3.17: Okapi接收到Ok-response:**

+ 由于module-list当中已经没有其他的module了，所以Okapi给login-module响应一个OK-response

**3.18: 此时 login-module可以看见是否接收到了用户的新权限，比如来自LDAP后端，在这种情况下，需要向权限模块发出请求来更新某些权限。它需要有一个权限才能做到这一点，而与Okapi的交互就像上面3.7 - 3.11中的db查找一样。**

**3.19: login-module返回一个携带新JWT的OK-resposne**

**3.20: Okapi发现没有更多的module需要调用了，给UI层返回OK-response**

**3.21:UI存储这个JWT，以供之后的操作**



## 开放性的问题和技术细节

+ 我们必须要注意，auth-module在生成newtoken的时候被访问了两次。auth-module将会接收两个一样的请求。一个简单的区别这两个请求的方式就是查看我们是否具有X-Okapi-Module-Permissions header。如果存在，那么这个请求就是一个普通的授权检查。如果不存在，服务路径(path)会指示出这是一个生成新token的请求，还是访问其他module的请求。
+ 在上述例子中，会有过多校验和错误处理
+ JWT属于auth-module，并且只有auth-module才可以创建 校验 标记它们，因为只有auth-module才具有`signing key`.不幸的是，这个key需要在集群中所有auth-module的运行节点共享。但是第一个启动的节点也许仍然会生成一个随机的key 并和剩下的节点共享，或者我们将这个值做硬编码处理(于每一个启动的节点)。
+ auth-module允许在JWT中随意添加一些东西。例如，它可以日志记录为目的保留某种sessionId，或者在原始请求到达时使用时间戳，这样它就可以跟踪请求的整个序列。



## Okapi Process

上述示例展示了在Okapi中程序的各种运行流程。这里我们总结一下Okapi所做的所有操作：

+ Okapi检查我们是否有一个 X-Okapi-Tenant的header，并且这个tenant的记录确实存在
+ 为这个tenant构建一个为本次请求服务的module-list，调用的顺序根据ModuleDescriptor中RoutingEntry的"level"大小而定
+ 这么module-list包含两个module，首先是auth-module，其次是实际要调用的module。但是我们很可能包含更多的module，也许包含一些监控日志的module(这种module在每次请求完成之后再去执行)
+ Okapi检查是否存在X-Okapi-Token。如果存在，则存储以供后续使用。实际上，Okapi将这个token复制进入module-pipeline(注：这个pipeline就是上一步中的module-list) 里面的每一个条目中。
+ Okapi浏览接下来需要调用的RoutingEntries，然后收集并记录这些服务所有的权限(reuired/desired permissions)。Okapi将这些权限进行除重操作，然后将它们放入额外的header中，即"X-Okapi-Permissions-Required"和"X-Okapi-Permissions-Desired"，用json格式数据进行存储(元素为string的list)，例如["some.perm","other.perm"]
+ Okapi浏览将要访问的modules的ModuleDescriptors，检查这些模块是否被授予了模块级权限。并将这些权限存入 X-Okapi-Module-Permissions header之中。将这些信息组织成Json格式，key为module-name，value是其对应的modulePermissions，例如:{ "motd" : [ "db.motd.read" ], "foo" : [ "bar.x", "bar.y" ] }
+ 现在Okapi可以开始对pipeline中的modules进行访问了，Okapi检查在pipeline中的module都在哪里运行
+ 传递最开始的HTTP请求到这个module，携带如下headers：
  + X-Okapi-Tenant: As it received it.
  + X-Okapi-Token: Either as it received it, or a new one from the auth module.
  + X-Okapi-Permissions-Required: Collected from the RoutingEntries.
  + X-Okapi-Permissions-Desired: Collected from the RoutingEntries.
  + X-Okapi-Module-Permissions: Collected from the ModuleDescriptions.
+ 当module作出了响应，Okapi检查这是否是OK-response，如果不是，那就将error-response返回给调用者，并结束本次调用流程。
+ 然后Okapi检查响应中是否携带着X-Okapi-Module-Tokens，这旨在检查module的认证授权工作是否完成，如果是认证完成的情况，那么Okapi会做如下的事情:
  + 查看是否存在假module `"_"`的token，如果存在，则在之后每一个module的调用中，携带着这个token
  + 查看是否存在已知module的tokens，然后为这个module复制这些tokens
  + 移除这些headers： X-Okapi-Permissions-Required，X-Okapi-Permissions-Desired，X-Okapi-Module-Permissions，这些headers已经完成了自己的使命。
+ 当Okapi执行完了pipeline中的所有modules之后，则返回最后一个响应给调用者



## The auth process

上述示例中展示了auth-module(authentication)做的各种事情，这里我们按照顺序总结一下。当auth-module接收到请求：

+ 检查我们拥有一个X-Okapi-Tenant header，这个检查在之前一定是被做过一遍，因为如果缺少了这个header，那么Okapi就会将本次请求fail掉，但是做一次校验总是好的。
+ 创建一个空的map，以供存储module tokens
+ 检查我们是否有X-Okapi-Token:
  + 签名和内容是否相符合
  + token中的tenant和X-Okapi-Tenant的内容是否相符，以便于让每次请求都能够指向正确的tenant
  + 如果任何一个校验不通过，那么就立即拒绝这次请求，并返回一个 code为400 的"Invalid Request"响应和一个可读的错误描述。
  + 如果有userId，则从token中提取
+ 然后auth-module检查X-Okapi-Permissions-Required 或 X-Okapi-Permissions-Desired中是否有值：
  + 如果存在userId，则从permission-module查询user的权限集，或者从缓存中读取信息(必须为有效缓存)
  + 检查JWT是否包含modulePermissions，如果包含，那么将这些permission添加到用户的permission-list当中
  + 对于每一个Permissions-Required的权限，auth-module在permission-list当中检查它是否存在。如果permission-list中没有Required权限，则拒绝这次请求，返回403 - Forbidden，和一个可读的错误信息(如果必要 则展示缺少的权限)。
  + 对于每一个Permissions-Desired的权限，auth-module会检查它在permission-list当中是否存在，如果存在，则在permission-list当中添加这个权限。
+ 接下来，auth-module检查JWT中是否存在modulePermissions，如果存在，则新建一个和原来的token类似的新token，但是缺少了modulePermissions信息，标记它，并以module map的形式存储到伪module `"_"`的信息当中。
+ 然后检查我们是否有X-Okapi-Module-Permissions。并且对于在该header中出现的所有module，auth-module校验了它们的module-name(只校验字母数字的module-name，不校验`"_"`)。然后auth-module获取了`"_"`module的JWT，将那些permissions添加到这个JWT当中，并进行标记，然后存储到map当中(key为module-name)
+ 它将module-token-map编码，并设置进X-Okapi-Module-Tokens，以json格式编码 内容格式大概如下:{"motd" : "xxx-motd-token-xxx", "db" : "xxx-db-token-xxx", "_" : "xxx-default-token-xxx"}。即使map当中没有内容，auth-module也会做这件事，目的在于告诉Okapi权限校验的工作已经完成。
+ 最后返回OK-response，携带如下headers:
  + X-Okapi-Permissions:
  + X-Okapi-Module-Tokens:







