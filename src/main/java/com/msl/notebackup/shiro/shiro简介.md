## 写在前面

安全一直是重中之重，作为Web开发人员，应当要学会利用正确的安全框架为自己的Application提供足够的保障。介于本人开发的项目大多以Spring为主，Netty为辅，所以本来想学Spring Security的，但是又看了一下[Shiro](https://shiro.apache.org/index.html)，对比起来，Shiro有以下优点：

```
1. 简单
2. 可适用于非Spring项目
```
遂决定先学习Shiro，待到技术成熟了，在考虑Spring Security也不迟。

## Shiro介绍

### Shiro特性

![](https://user-gold-cdn.xitu.io/2020/4/15/1717c8ca0a1c70ff?w=491&h=253&f=png&s=39340)

#### Authentication

验证。类似Web应用中的"登录"操作，确定用户身份。

#### Authorization

授权。Access Control，访问控制；限定已登录的用户的行为，比如是否可以访问某个网页。

#### SessionManager

管理特定于用户的Session，即使是在非Web应用也可以。

#### Cryptography

密码学，提供密码支持，确保数据源使用加密算法，同时保持易用性。

## Shiro体系结构

### 更高级别的概览

![](https://user-gold-cdn.xitu.io/2020/4/15/1717c92c98c3e1cd?w=410&h=220&f=png&s=37986)

#### Subject

主体(Subject)是一个特定于安全的当前执行某个操作的用户的"视图"。"用户"并不一定是真人，也可以是第三方服务，或其他的应用。

subject实例全部被绑定到一个SecurityManager，当你与subject交互时，实际上是和SecurityManager进行交互的。

#### SecurityManager

Shiro的核心部分，它像一个"保护伞"，组织所有内部的组件完成安全操作。一旦一个SecurityManager被配置到一个应用上，那么就随它跑了，开发者可以把大部分的精力放在与subject的交互上了。

记得，当你与subject交互时，实际上是SecurityManager在处理实际的操作。

#### Realms

域。Realms作为桥接Shiro和应用安全数据(就是数据库中需要安全保护的数据)的对象。当实际需要与安全数据打交道时，Shiro会使用Realms来处理。你可以把Realms当成一个特定于安全的DAO组件，它封装了实际的数据源，并在Shiro需要时为其提供数据，当配置认证/授权时，你必须至少指定一个这样的DAO组件。

Shiro还提供了脱离于框架的Realms连接，通过此种方式，你可以注入任何你想要的数据源连接。

就像其他组件那样，SecurityManager管理Realms来实现对于确认安全身份数据的访问和使用。

### 更细节化的结构层次

![](https://user-gold-cdn.xitu.io/2020/4/15/1717cc565290ed1c?w=525&h=438&f=png&s=163490)

#### Subject

当前与应用进行交互的实体的特定于安全的"视图"。(好拗口，看原文：A security-specific ‘view’ of the entity (user, 3rd-party service, cron job, etc) currently interacting with the software.)

#### SecurityManager

Shiro框架的核心，管理并调度者内部的组件，它同样也管理Shiro的对于每个用户的"视图"，所以它知道怎么为每个用户提供安全操作。

#### Authenticator

认证器，作用于试图登录的用户操作。认证器还知道怎么选取Realm来获取这个用户的信息。

#### Authentication Strategy

如果不止一个Realm被配置了，那么AuthenticationStrategy会联合多个Realm来实行具体的认证策略。

#### Authorizer

授权器。针对指定用户，提供其可访问的域，同认证器一致，可以访问后端数据，然后进行处理访问控制。

#### SessionManager

为session提供加强版的会话操作，甚至可以在非Web领域使用。它会尝试使用已存在的session机制，比如Servlet容器。如果没有，它会提供内置的机制来达到使用效果。

#### SessionDao

代表SessionManager来进行持久化操作(CRUD)，或注入外在的数据源。

#### CacheManager

缓存管理。负责管理被Shiro组件使用的缓存，因为很多访问操作都会优先访问缓存，所以，缓存的好坏很大程度上影响了程序的性能。任何开源的缓存框架都可以被注入到Shiro里面去。

#### Cryptography

Shiro封装了很多易用且易理解的密码加密解密，哈希，编解码实现类。Shiro旨在然后所有智商正常的凡人轻易地使用密码学(不是我说的，官网确实是这么介绍的)。

#### Realms

桥接Shiro与数据库的桥梁。

### SecurityManager

SecurityManager和组件使用Java Bean编辑方式，那些使用Bean管理的框架可以直接使用Shiro的Bean。

### 设计


## Shiro术语

#### Authentication

是一个验证的过程——确认subject的身份，以确保与其进行相关的互动。

#### Authorization

又称为访问控制，是一个决定用户/subject可以做什么，或访问什么资源的过程。

#### Cipher

密码，是一个用于加密或解密的算法。这个算法基于一个被称为key(键)的信息，加密因key而异，所以没有key的话，进行解密是很困难的。密码有很多，比如，作用于信号块的块密码通常大小固定。而流密码作用于一个连续的流信号。对称密码使用同一个key进行加密解密，而非对称密码的加密解密使用不同的key。如果一个key没法从另一个密码里推导出来，那么可以创建可以被共享的公/私键值对。

#### Credential

凭据。凭据是一段信息，这段信息可以用来验证用户/subject的身份。通常情况下，在认证期间，一个凭据是和一个准则(principal)一起被呈现的。凭据一般情况下都是秘密性的信息，只有某一确定的用户/subject知道，比如密码，PGP，生物属性，或类似的机制。

设计初衷是，对于某个准则，只有一个人知道和它准确对应的凭据。对于系统而言，如果某个用户/subject可以提供一个与储存在系统里面的准侧匹配的凭据的话，就可以认证这个用户/subject是可以使用的，用户/subject被信任的程度取决于凭据的安全程度(比如，生物信息>密码)。

#### Cryptography

密码学。密码学的意义是，保护信息以防止被意料之外的行为访问，比方说通过隐藏或转换成别的似乎看起来没意义的内容。Shiro使用两个主要的方法进行密码学处理：使用公开或私有键进行密码加密，比如邮箱地址；使用哈希进行不可逆的加密，比如密码。

#### Hash

哈希。哈希函数用于进行单向的，不可逆的对输入源(有时称为消息)的加密。加密成编码的哈希值(有时被称为信息摘要)。时常用于密码，数字指纹，或者底层比特数组。

#### Permission

许可。正如Shiro描述的那样，仅仅是一个描述功能性的语句。它们是安全策略最底层的实现，它们仅仅定义应用能做"什么"，而不能定义"谁"能进行操作。许可仅仅是行为的语句，举例：

```
1. 打开文件
2. 访问'/user/list'网页
3. 打印文档
4. 删除'Jhon'用户
```

#### Principal

准则可以是应用用户/subject的任何身份属性。只要可以对应用产生意义，那就是可以的，比如用户名，姓名，暂定名，社会安全编码(身份证号)，user ID等。

Shiro指出每个应用用户都有一个"主准则"，它可以是任意的属性，但必须是唯一的，最好对应数据库中的主键。

#### Realm

把它理解成特定于安全数据的DAO组件。可以访问你的特定于应用的数据，然后返回给Shiro，这样，不管你配置了几个数据源，又或者不管你的数据库操作怎么写，Shiro都能提供统一的API用于实现简单易理解的操作。

Realm通常拥有一对一的数据源对应关系，Realm接口的实现类，使用特定于源的数据API来查找认证数据(角色，许可等)，比如JDBC，文件IO，Hibernate或者JPA，或者任何其他的数据访问API。

#### Role

对于角色的定义，可以说是很模糊的。Shiro这么认为：Role是许可的集合，它包含了多个许可声明，若是按照Shiro给出的定义来编写你的应用，嚯嚯，你会发现在控制安全策略方面，你会更加得心应手。

#### Session

Session是用户/subject与应用交互期间所产生的数据上下文。一个用户/subject对应一个Session。数据可以被从Session中添加，读取，移除。也可以在未来某个时间，由应用再次使用。当用户退出或不活跃超时的时候，Session会被销毁。

类似于HTTP Session，Shiro的Session也可以实现类似功能，也可以被用于任意环境，不仅限于Servlet容器或EJB容器。

#### Subject

对于那些与应用进行交互的事物的抽象，比如说一个Subject可以代表一个人，也可以代表一个程序，服务，其他的系统等。

## Shiro引用文档

### 核心

#### Authentication(验证)

Shiro的验证基于Subject，这点前面提到了。说一下验证步骤吧：
```
1. 收集Subject的Principal(一般是用户名)和Credential(一般是密码)
2. 提交收集来的Principal和Credential用以验证
3. 判断验证结果
```
第一步，收集验证结果，这里以Username-Password这种常见的方式为例：
``` Java
// 注意哈，至于username和password是怎么来的，都无所谓
// 可以是HTML表单提交(后端开发最常见的)，也可以是文件读取
UsernamePasswordToken token = new UsernamePasswordToken(username, password);

// 是否开启“记住我”模式，这个模式后面会纤细说明
token.setRememberMe(true);
```
第二步，提交收集结果进行验证：
``` Java
Subject currentUser = SecurityUtils.getSubject();

currentUser.login(token);
```
第三步，处理验证结果，因为如果验证失败了，那是会抛异常的，所以应该使用try-catch语句：
``` Java
try {
    currentUser.login(token);
} catch ( UnknownAccountException uae ) { ...
} catch ( IncorrectCredentialsException ice ) { ...
} catch ( LockedAccountException lae ) { ...
} catch ( ExcessiveAttemptsException eae ) { ...
} ... catch your own ...
} catch ( AuthenticationException ae ) {
    //unexpected error?
}
// 到这里说明验证通过了
```

来看一下具体的验证过程：

(只看高亮的)

![](https://user-gold-cdn.xitu.io/2020/4/26/171b47aa4badfe04?w=479&h=310&f=png&s=90539)

说直白点就是：


![](https://user-gold-cdn.xitu.io/2020/4/26/171b48a69472423e?w=2732&h=2048&f=png&s=1486747)

这里的验证器是可以自定义的，验证策略也是可以自己实现的。一个验证策略会在一个验证活动中被访问四次，分别是：在任一域被调用之前；立即地在一个独立的域的getAuthenticationInfo()被调用之前；立即地在一个独立的域的getAuthenticationInfo()被调用之后；在所有的域被调用之后。验证策略也会负责整合Realm的执行结果并最终返回一个AuthenticationInfo实例用来指出验证的最终结果。

Shiro默认有三个验证策略：AtLeastOneSuccessfulStrategy, FirstSuccessfulStratrgy(仅使用第一个域的返回值作为结果), AllSuccessfulStrategy。默认使用AtLeastOneSuccessfulStrategy。

来看一下“记住我”和“已验证”的不同。说白了就是，“已验证”的身份确认度更高，“记住我”操作仅适用于普通交互，对于关键性交互，比如支付，修改信息，记得再次要求进行验证，而不能使用“记住我”。

当然，在用户交互完，记得调用subject.logout()方法来登出用户，同时记得把用户重定向到新的界面，不然没法清空Cookie。
#### Authorization(授权)

授权，通常被称为访问控制。用于实现对于用户访问的控制。(包含三个主要的组件：Permissions, Roles, Users)

**许可(Permissions)**。许可是Shiro安全策略最核心的部分。指出了一个Subject可以干嘛（执行什么行为，比如打开文件，向数据库中添加数据等）。对于许可，我们建议，最好把它们设计成基于资源(Resources)和行为(Actions)的。许反映的是"行为表现(behavior)"，请记住，它们没反映"用户"。

许可也可以使Role进行封装，也可以直接关联到某一（些）具体的User(Subject)，或某一些封装在"组"里面的User。

**角色(Role)**。
角色是一个实体，包含了具体的行为和职责。角色可以被指派到某些用户账户上，这样，用户可以"做"或"不做"角色属性所指明的行为。

有两种角色，隐式角色和显式角色，显式角色可以更加细粒度的控制应用的访问控制，我们(不是我，是Shiro)推荐使用显式角色。

这里有一个基于资源的访问控制的[文章](https://stormpath.com/blog/new-rbac-resource-based-access-control)，推荐一读。

**用户(User)**。
用户实际上是与应用交互的"谁"，也就是Subject。在Shiro里面，这个就是Subject。你的应用数据模型定义了用户可以执行的行为。当然指派用行为的方式和前面提到的一样——通过直接把许可绑定到用户，把包含了许可的角色指派到用户或包含了用户"组"。

Shiro使用封装了数据获取细节的Realm实现类来实现数据交互。

**验证Subject**

验证Subject的方法有两个（其实是三个，但是第三个不常用）：程序化授权（就是编写if-else语句进行处理，比方说：如果密码==XXX，else...）或基于JDK的注解验证。这两个方式


**程序化授权**

来看看程序化授权的使用, 它包含两个方式: 基于角色(Role)的授权和基于许可(Permission)的授权：

**1. 基于角色的授权**

**1.1. Role检查**
来看典型的用法吧！
``` Java
Subject currentUser = SecurityUtils.getSubject();

if (currentUser.hasRole("administrator")) {
    //show the admin button 
} else {
    //don't show the button?  Grey it out? 
}
```

|方法名|用法|
|:-------|:-------|
|boolean hasRole(String roleName) | 根据角色名判断是否包含此角色|
|List\<Boolean\> hasRoles(List\<String\>) | 返回一个boolean值列表，对应每一个角色名|
|boolean hasAllRoles(Collection\<String\>) roleNames | 只有包含全部角色时才返回真|

**1.2. 角色断言**
看一下用法:
``` Java
Subject currentUser = SecurityUtils.getSubject();

//guarantee that the current user is a bank teller and 
//therefore allowed to open the account: 
currentUser.checkRole("bankTeller");
openBankAccount();
```
如果断言失败(不包含这个角色),会抛出异常,否则程序正常执行

方法名 | 用法
|:------|:------|
checkRole(String RoleName) | 如果包含,正常执行下去, 否则抛异常
checkRoles(Collection\<String\> roleNames) | 只有全部包含才会正常执行,否则抛异常
checkRoles(String... roleNames) | 作用同上, 但是入参不同

**2. 基于许可的授权**

**2.1. 许可检查**
许可检查有两种方式，Permission类对象作为参数，Permission名作为参数。

**2.1.1. 类对象作为参数**

此种方法使用一个实现了org.apache.shiro.authz.Permission接口的类对象作为参数。相比使用String作为参数的方式，这种可以实现更加细粒化的控制，同时可以更加准确的确定你需要的（Permission）类型。

看一下用法：
``` Java
Permission printPermission = new PrinterPermission("laserjet4400n", "print");

Subject currentUser = SecurityUtils.getSubject();

if (currentUser.isPermitted(printPermission)) {
    //show the Print button 
} else {
    //don't show the button?  Grey it out?
}
```
可用的方法：
方法名 | 描述
|:-------|:-------|
isPermitted(Permission p) | 如果Subject被允许执行许可所包含的行为，返回真，否则返回false
isPermitted(List<Permission> perms) | 返回一个与参数顺序对应的结果列表
isPermittedAll(Collection<Permission> perms) | 只有全部满足才会返回真

**2.1.2. 名称作为参数**

使用String作为参数的用法，比使用对象作为参数的方法更加容易编写，但是相应的，失去了类型安全检查，还有更加细致化的自定义许可也不可使用。

看一下用法：
``` Java
Subject currentUser = SecurityUtils.getSubject();

if (currentUser.isPermitted("printer:print:laserjet4400n")) {
    //show the Print button
} else {
    //don't show the button?  Grey it out? 
}
```
上述代码几乎是下面这个的捷径版本：
``` Java
Subject currentUser = SecurityUtils.getSubject();

Permission p = new WildcardPermission("printer:print:laserjet4400n");

if (currentUser.isPermitted(p) {
    //show the Print button
} else {
    //don't show the button?  Grey it out?
}
```
实际使用中，常常使用基于':'(冒号)分割的字符串，这在org.apache.shiro.authz.permission.WildcardPermission里有定义，也是使用String作为参数时常用的类

再看一下方法列表：
方法名 | 描述
|:----|:---------------|
isPermitted(String perm) | 判断是否被允许执行某一许可
isPermitted(String... perms) | 返回结果列表
isPermittedAll(String... perms) | 只有全部满足才返回true

**2.2. 许可断言**
同基于角色的用法一样，断言失败抛异常，否则继续执行：
``` Java
Subject currentUser = SecurityUtils.getSubject();

//guarantee that the current user is permitted 
//to open a bank account: 
Permission p = new AccountPermission("open");
currentUser.checkPermission(p);
openBankAccount();
```
或使用String作为参数的断言：
``` Java
Subject currentUser = SecurityUtils.getSubject();

//guarantee that the current user is permitted 
//to open a bank account: 
currentUser.checkPermission("account:open");
openBankAccount();
```
直接看方法列表吧！

方法名 | 描述
checkPermission(Permission p) | 被允许就返回true
checkPermission(String perm) | 同上
checkPermissions(Collection\<Permission\> perms) | 当所有的Permission均被允许才返回true
checkPermissions(String... perms) | 同上

**基于注解的授权**
在使用此方法之前，请确保你配置了AOP代理。对于不同的代理，有不同的方法，这一点很遗憾，Shiro没有统一。

这里看不同的启用AOP的例子

[AspectJ](https://github.com/apache/shiro/tree/master/samples/aspectj)

[Spring](https://shiro.apache.org/spring.html)

[Guice](https://shiro.apache.org/guice.html)

**RequiresAuthentication**注解

这个注解限制了被标注的方法只能被已认证的Subject访问。
``` Java
@RequiresAuthentication
public void updateAccount(Account userAccount) {
    //this method will only be invoked by a
    //Subject that is guaranteed authenticated
    ...
}
```
等同于这个：
``` Java
public void updateAccount(Account userAccount) {
    if (!SecurityUtils.getSubject().isAuthenticated()) {
        throw new AuthorizationException(...);
    }

    //Subject is guaranteed authenticated here
    ...
}
```
**RequiresGuest**

这个注解启用访客模式，只有**没有**验证或**没有**“记住我”的Subject才可以访问此方法
``` Java
@RequiresGuest
public void signUp(User newUser) {
    //this method will only be invoked by a
    //Subject that is unknown/anonymous
    ...
}
```
等同于：
``` Java
public void signUp(User newUser) {
    Subject currentUser = SecurityUtils.getSubject();
    PrincipalCollection principals = currentUser.getPrincipals();
    if (principals != null && !principals.isEmpty()) {
        //known identity - not a guest:
        throw new AuthorizationException(...);
    }

    //Subject is guaranteed to be a 'guest' here
    ...
}
```
**RequiresPermissions**

此注解标注的方法指明了Subject必须至少拥有一个许可才可以执行它的逻辑
``` Java
@RequiresPermissions("account:create")
public void createAccount(Account account) {
    //this method will only be invoked by a Subject
    //that is permitted to create an account
    ...
}
```
等同于：
``` Java
public void createAccount(Account account) {
    Subject currentUser = SecurityUtils.getSubject();
    if (!subject.isPermitted("account:create")) {
        throw new AuthorizationException(...);
    }

    //Subject is guaranteed to be permitted here
    ...
}
```

**RequiresRoles**

这个注解指出访问此方法的Subject必须拥有参数内的全部角色，否则会抛出异常
``` Java
@RequiresRoles("administrator")
public void deleteUser(User user) {
    //this method will only be invoked by an administrator
    ...
}
```
等同于：
``` Java
public void deleteUser(User user) {
    Subject currentUser = SecurityUtils.getSubject();
    if (!subject.hasRole("administrator")) {
        throw new AuthorizationException(...);
    }

    //Subject is guaranteed to be an 'administrator' here
    ...
}
```

**RequiresUser**

因为Shiro的User就是Subject，所以这里要求操作的Subject必须是**已验证或已记住**的。

``` Java
@RequiresUser
public void updateAccount(Account account) {
    //this method will only be invoked by a 'user'
    //i.e. a Subject with a known identity
    ...
}
```
等同于：
``` Java
public void updateAccount(Account account) {
    Subject currentUser = SecurityUtils.getSubject();
    PrincipalCollection principals = currentUser.getPrincipals();
    if (principals == null || principals.isEmpty()) {
        //no identity - they're anonymous, not allowed:
        throw new AuthorizationException(...);
    }

    //Subject is guaranteed to have a known identity here
    ...
}
```

**授权序列**

看一些实际的验证过程吧！

数字代表具体的步骤(老规矩，只看高亮)
![](https://user-gold-cdn.xitu.io/2020/4/28/171bfcbf8154a271?w=479&h=310&f=png&s=83959)

第一步：程序调用Subject的任意的hasRole*, checkRole*, isPermitted*, 或 checkPermission*方法变种。

第二步：Subject实例，其实更多是代理给SecurityManager的DelegatingSubject，然后调用SecurityManager的同名方法变种，SecurityManager实现了org.apache.shiro.authz.Authorizer接口，这个接口定义了一些特定于授权的方法。

第三步：SecurityManager会通过authorizer的相应的方法转发或代理给内部的org.apache.shiro.authz.Authorizer实例。authorizer实例实际上是一个ModularRealmAuthorizer，它支持整合多个Realm然后执行授权。

第四步：每个被配置的Realm都会被扫描一遍，以来查看他们有没有实现同一个Authorizer接口，如果是，那么Realm的同名方法那些hasRole*, checkRole*, isPermitted*, 或 checkPermission*会被调用

**ModularRealmAuthorizer**

正如前面提到的，SecurityManager实际上是通过它的ModularRealmAuthorizer来实现业务逻辑的。

对于每一个授权操作，ModularRealmAuthorizer会迭代它的Realm集合，然后根据**迭代顺序**进行处理，处理逻辑如下：
```
1. 如果Realm实现了这个Authorizer接口，那么会进行下一步——调用Realm的同名方法(hasRole*, checkRole*, isPermitted*, 或checkPermission*)：
    1.1. 如果Realm的方法抛异常，异常会被传播到Subject，迭代访问终止
    1.2. 如果返回true，程序终止，返回true，这是一种安全策略(雾)
2. 否则直接忽略
```

对于配置全局PermissionResolver

当使用基于String的许可检查时，大多数的Realm实现类会把String转换成一个Permission实例。这是因为许可被假设为使用隐式逻辑而不是相等检查。对于此，Shiro会使用PermissionResolver来进行处理，它会把你的String表达式翻译成Permission实例。对于这个，Shiro使用WildcardPermissionResolver来进行默认支持，注意，这个类仅支持WildcardPermission类的String表达式，如果你想自定义自己的PermissionResolver，记得配置Realm。

注意，如果你想配置Realm来接受自定义PermissionResolver的话，记得让Realm实现PermisionResolverAware接口。或者显式地使用PermissionResolver设置Realm。

同样，还有一个**RolePermissionResolver**，它可以把一个角色名转换成一个许可集合，然后供Realm使用。你可以配置全局RolePermissionResolver来进行对所有的Realm进行配置，当然别忘了让Realm实现RolePermisionResolverAware接口。或者显式地为单个Realm设置RolePermissionResolver实例。

#### Realm(域)

首先一点，不推荐隐式指派，就是对于向SecurityManager设置域时，最好显式地设置，这样也可以指明他们的顺序，要知道，Realm的顺序在认证和授权里尤其重要。

在Realm的实际认证方法：getAuthenticationInfo(token)被调用之前；它的支持方法会被调用以判断传进来的token是否可以被处理，如果此Realm支持处理此格式的token，那么才会进一步进行实际的认证。

来看一个典型的验证过程：
```
1. 检查传入的token
2. 根据传进来的principal(用户名之类的)，从数据源(比如MySQL)中获取用户信息
3. 对比token的credential
4. 若是匹配，返回一个AuthenticationInfo实例，这个实例封装了用户信息，以便Shiro理解
5. 若是不匹配，抛一个异常
```

小提示：直接实现Realm方法可能费时费力不讨好，可以试着实现AuthorizingRealm抽象类，它帮你实现了大多数基本的方法。

对于进行credential验证，可以试着使用CredentialsMatcher来进行处理，它作为一个匹配器存在，Shiro提供了简单的CredentialsMatcher实现，比如SimpleCredentialsMatcher和HashedCredentialsMatcher。

对于他们的区别。

**SimpleCredentialsMatcher**提供了简单的对比策略(比如字符串匹配)，它可以对比String, character数组, byte数组, 文件和输入流。

**HashedCredentialsMatcher**对credential进行单向哈希，这相比于普通的直接保存对比纯密码来说，多了很多安全性。Shiro为HashedCredentialsMatcher提供了很多细粒化的子类实现。

来看一种简单的HashedCredentialsMatcher用法：
``` Java
import org.apache.shiro.crypto.hash.Sha256Hash;
import org.apache.shiro.crypto.RandomNumberGenerator;
import org.apache.shiro.crypto.SecureRandomNumberGenerator;
...

// 使用SHA-256加密算法
RandomNumberGenerator rng = new SecureRandomNumberGenerator();
Object salt = rng.nextBytes(); // 随机数作为"盐"

// 指明哈希迭代次数，编码形式，"盐"值
String hashedPasswordBase64 = new Sha256Hash(plainTextPassword, salt, 1024).toBase64();

User user = new User(username, hashedPasswordBase64);

// 把"盐"保存起来，后期的验证会用到
user.setPasswordSalt(salt);
userDAO.create(user);
```

最后，你应该确保你的Realm实现返回一个SaltedAuthenticationInfo实例而不是普通的AuthenticationInfo，SaltedAuthenticationInfo可以确保HashedCredentialsMatcher可以获取到你的"盐"值。

如果你想关闭验证，仅需要把Realm的支持方法返回值全部设为false就行。

对于授权，分为两种：基于角色的授权和基于许可的授权

**基于角色的授权**
当在Subject上调用hasRoles或checkRoles方法的重载方法之一时：
```
1. Subject把实际操作代理给SecurityManager来确定给定的角色是否被指派。
2. SecurityManager把请求代理给Authorizer。
3. 逐个迭代授权域(Authorizing Realm)来验证角色是否被指派给了这个Subject，如果所有的授权域都没有表示这个角色被指派给了这个Subject，那么返回false以表示拒绝访问。
4. 授权域的AuthorizationInfo的getRoles()方法来获取指派给这个Subject的全部角色。
5. 如果等待验证的角色在角色列表(getRoles()方法的返回值)里找到了，那么返回true。
```

**基于许可的验证**
当任何isPermitted()或checkPermission()方法的重写方法在Subject里被调用的话：
```
1. Subject会把检查许可这个任务代理给SecurityManager来完成。
2. SecurityManager会代理给Authorizer。
3. 遍历全部域，若所有的域都没有授权此许可，Subject的访问被禁止
4. 授权域使用以下顺序来进行许可检查：
    4.1. 首先，它通过在AuthorizationInfo上调用getObjectPermissions()和getStringPermissions()方法并汇总结果来直接标识分配给Subject的所有权限。
    4.2. 如果注册了RolePermissionResolver，则可通过调用RolePermissionResolver.resolvePermissionsInRole()来基于分配给Subject的所有角色检索Permissions。
    4.3. 对于1和2的结果汇总，会调用implies()方法来检查这些许可是否隐含已检查的许可。
```

#### Session Management(会话管理)

**启用Session**

在Shiro里面使用Session非常简单，仅需这样做：
``` JAVA
Subject currentUser = SecurityUtils.getSubject();

Session session = currentUser.getSession();
session.setAttribute( "someKey", someValue);
```
注意到getSession()方法，如果方法参数是true，那么会在没有Session时创建Session，如果为false，那么会返回空，默认true。

**SessionManager**

SessionManager，负责一个应用中的Session的创建，删除，更新，查找操作。默认情况下Shiro会使用DefaultSessionManager来进行操作。通过SecurityManager来设置SessionManager。

1. **设置Session超时时间**

默认超时时间为1小时，可以通过设置SessionManager的globalSessionTimeout属性来设置新的超时时间，单位：ms。

2. **设置Session监听器**

你可以通过设置监听器来对状态发生改变的Session做出响应（比如Session被创建）。SessionManager的sessionListeners属性是一个集合，里面包含了全部的Session监听器。通过实现SessionListener接口或拓展SessionListenerAdapter类。

``` JAVA
package org.apache.shiro.session;

public interface SessionListener {

  
    void onStart(Session session);

   
    void onStop(Session session);

   
    void onExpiration(Session session);
}

```

3. **Session储存**

Session可以存储在硬盘，内存，关系型数据库，NoSQL等里面，Session的CRUD由SessionDAO负责。你可以实现自己的SessionDAO实现并通过SessionManager来设置。
``` JAVA
package org.apache.shiro.session.mgt.eis;

import org.apache.shiro.session.Session;
import org.apache.shiro.session.UnknownSessionException;

import java.io.Serializable;
import java.util.Collection;

public interface SessionDAO {

    Session readSession(Serializable sessionId) throws UnknownSessionException;

    void update(Session session) throws UnknownSessionException;

    void delete(Session session);

    Collection<Session> getActiveSessions();
}

```
就很明显，增删改查嘛～

但是有一点，如果你写的是Web应用，那么你不能使用DefaultSessionManager而是DefaultWebSessionManager，然后才能设置SessionDAO。

3.1. **EHCache SessionDAO**

强烈建议开启EHCache，因为它会在内存吃紧时自动进行硬盘存储，保证了数据的完整性。在你不自定义SessionDAO的情况下，EHCache除了适用于会话操作，还适用于所有的缓存操作。如果你需要基于容器的集群机制，它也是一个不错的选择。

通过把SecurityManager的cacheManager属性设为EHCache属性即可完成启用。

Web应用一般使用容器自带的Session管理Session，所以SessionDAO可能不会起作用，所以应重新配置SessionManager来完成启用。

如果你想通过EHCacheManager来设置EHCache的话，你可能会用到ehcache.xml，核心配置如下所示：
``` XML
<cache name="shiro-activeSessionCache"
       maxElementsInMemory="10000"
       overflowToDisk="true"
       eternal="true"
       timeToLiveSeconds="0"
       timeToIdleSeconds="0"
       diskPersistent="true"
       diskExpiryThreadIntervalSeconds="600"/>
```
其中请确保overflowToDisk和eternal为true，后者确保缓存不会被意外关闭。

3.2. **自定义Session的ID**

默认使用JDK的UUID工具来生成Session的ID，通过实现SessionIdGenerator接口来自定义SessionID生成策略。

4. **Session验证和定时检验**

Session会在每次访问时进行验证(查看是否过期需要删除)，但是对于长时间不访问的Session，可能就会一直没法验证，遂需要一个定时器每隔一段时间检查session存储看有没有过期的，把它删了。默认一个小时检查一次。SessionValidationScheduler是默认的定时器。通过SessionValidationScheduler的interval属性，即可设置间隔。然后通过SessionManager的sessionValidationScheduler属性来设置定时器。

对于某些特殊情况，你可以选择在即使验证失败后依旧可以保留Session，但是你依旧得记得手动删除！这是很重要的，还有，禁用定时器验证和禁用删除是两码事，禁用了定时器验证之后你必须做一个自己的定时器来完成类似的工作。

**Session集群**

[这里](https://shiro.apache.org/session-management.html#session-clustering)

**Session和Subject状态**

Session可以用于保存用户验证信息和验证状态。这在"记住我"层面是很有用的。呃，还有一个忘说了，就是可以使用Subject Subject = new Subject.Builder().sessionId(sessionId).buildSubject();来根据sessionId来构建Subject。

通过SecurityManager.subjectDAO.sessionStorageEvaluator.sessionStorageEnabled属性设为false来禁用Subject状态会话存储。

通过SessionStorageEvaluator接口还可以指定针对不同访问者是否开启Session机制
``` JAVA
public interface SessionStorageEvaluator {

    public boolean isSessionStorageEnabled(Subject subject);

}
```
典型用法如下：
``` JAVA
public boolean isSessionStorageEnabled(Subject subject) {
        boolean enabled = false;
        if (WebUtils.isWeb(Subject)) {
            HttpServletRequest request = WebUtils.getHttpRequest(subject);
            //set 'enabled' based on the current request.
        } else {
            //not a web request - maybe a RMI or daemon invocation?
            //set 'enabled' another way...
        }

        return enabled;
    }
```

#### Cryptography(密码学)
### Web应用

### 附加支持

### 整合


## END