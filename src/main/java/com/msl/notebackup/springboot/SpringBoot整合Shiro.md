### 给自己的Web框架添加合适的安全框架是一件很重要的事，在这里，我选择了Shiro，因为它更加简单且通俗易懂，现在来看看怎么把它和SpringBoot整合到一块吧！

关于Shiro基本原理，可以看[这个](https://juejin.im/post/6844904143174238215)。

## 核心依赖
``` XML
<dependency>
    <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring-boot-web-starter</artifactId>
    <version>${your.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-ehcache</artifactId>
    <version>${your.version}</version>
</dependency>
```
在这里还要添加数据库，mybatis，web等依赖

## 数据库创建
1.sys_user
``` SQL
CREATE TABLE `sys_user` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `username` varchar(25) DEFAULT NULL,
  `password` varchar(255) DEFAULT NULL,
  `salt` varchar(25) DEFAULT NULL,
  `state` int DEFAULT '0',
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b727668953a9e?w=864&h=190&f=png&s=30078)
对应的POJO
``` JAVA
public class SysUser {

    private Long id;

    private String username;

    private String nickname;

    private String password;

    private String salt;

    private Integer state;
}
```
2.sys_role
``` SQL
CREATE TABLE `sys_role` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(25) DEFAULT NULL,
  `desc` varchar(25) DEFAULT NULL,
  `is_available` tinyint(1) DEFAULT '0',
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b729b1b552abe?w=897&h=167&f=png&s=26338)
对应的POJO
``` JAVA
public class SysRole {

    private Long id;

    /**
     * 角色名
     */
    private String name;

    private String desc;

    private Boolean isAvailable = Boolean.FALSE;
}
```
3.sys_permission
``` SQL
CREATE TABLE `sys_permission` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(25) DEFAULT NULL,
  `desc` varchar(25) DEFAULT NULL,
  `resource_type` varchar(25) DEFAULT NULL,
  `url` varchar(25) DEFAULT NULL,
  `parent_id` bigint DEFAULT NULL,
  `parent_ids` varchar(25) DEFAULT NULL,
  `is_available` tinyint(1) DEFAULT '0',
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b72bcc1f2ed48?w=910&h=256&f=png&s=44496)
对应的POJO
``` JAVA
public class SysPermission {

    private Long id;

    /**
     * 许可名
     */
    private String name;

    /**
     * 资源类型
     */
    private String resourceType;

    /**
     * 资源url
     */
    private String url;

    /**
     * 许可字符串
     */
    private String desc;

    /**
     * 父编号
     */
    private Long parentId;

    private String parentIds;

    private Boolean isAvailable = Boolean.FALSE;
}
```
4.sys_user_roles
``` SQL
CREATE TABLE `sys_user_roles` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint DEFAULT NULL,
  `role_id` bigint DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b72d3253f8b10?w=911&h=146&f=png&s=21483)
对应的POJO
``` JAVA
public class SysUserRoles {

    private Long id;

    private Long userId;

    private Long RoleId;
}
```
5.sys_user_permissions
``` SQL
CREATE TABLE `sys_user_permissions` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint DEFAULT NULL,
  `permission_id` bigint DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b72edd35d26d9?w=909&h=147&f=png&s=21933)
对应的POJO
``` JAVA
public class SysUserPermissions {

    private Long id;

    private Long userId;

    private Long permissionId;
}
```
6.sys_role_permissions
``` SQL
CREATE TABLE `sys_role_permissions` (
  `id` int NOT NULL AUTO_INCREMENT,
  `role_id` bigint DEFAULT NULL,
  `permission_id` bigint DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```
![](https://user-gold-cdn.xitu.io/2020/6/15/172b730f5b780bb8?w=910&h=147&f=png&s=21378)
``` JAVA
public class SysRolePermissions {

    private Long id;

    private Long roleId;

    private Long permissionId;
}
```

**注意哈！我是图简单，数据类型全用最简单的了，实际设计请根据需要设计！！！**

## 基本目录结构以及设计

![](https://user-gold-cdn.xitu.io/2020/6/15/172b737c0f47c3d9?w=517&h=1829&f=png&s=177599)

### ShiroConfig
此类提供shiro的基本配置，比如ShiroFilterChainDefinition：用来定义拦截路径，这不同于集成SpringMVC的样子，登录路径以及登录成功，未验证的路径是在配置文件里设置的。
``` JAVA
package com.learn.shiro.config;

import com.learn.shiro.dao.shiro.WebRealm;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.cache.CacheManager;
import org.apache.shiro.cache.ehcache.EhCacheManager;
import org.apache.shiro.session.mgt.SessionManager;
import org.apache.shiro.spring.web.config.DefaultShiroFilterChainDefinition;
import org.apache.shiro.spring.web.config.ShiroFilterChainDefinition;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

/**
 * @author SuanCaiYv
 * @time 2020/6/9 下午4:27
 */
@Configuration
public class ShiroConfig {

    /**
     * 注入域对象，用来完成验证和授权操作
     */
    @Autowired
    private WebRealm webRealm;

    /**
     * 进一步根据访问路径配置更详细的过滤器，add()的第二个参数代表了过滤器的名字
     * 来看详细的解释
     * anon: AnonymousFilter，允许不做验证直接访问，相当于不加过滤器
     * authc: FormAuthenticationFilter，要求请求的用户必须是已验证的，否则强制重定向到设置好的登录界面
     * authcBasic: BasicHttpAuthenticationFilter，要求org.apache.shiro.subject.Subject#isAuthenticated()返回真，否则要求登录
     * authcBearer: BearerHttpAuthenticationFilter，和上面效果差不多，协议种类不同
     * logout: LogoutFilter，对当前用户进行登出操作，并重定向到设定好的“重定向Url”
     * noSessionCreation: NoSessionCreationFilter，禁用Session的创建，对于可能产生REST，SOAP等交互结果的操作很有用
     * perms: PermissionsAuthorizationFilter，如果当前用户拥有map指定的值时，允许访问
     * port: PortFilter把请求限定在某一指定端口，如果请求不在此端口发起请求，那么重定向到这个端口
     * rest: HttpMethodPermissionFilter，把HTTP请求方法转换成一个行为值，并用这个值构建一个许可用来进行授权验证
     * roles: RolesAuthorizationFilter，当当前用户持有map包含的值之一时，允许访问，如果持有的所有角色都没被包含，拒绝访问
     * ssl: SslFilter，如果是SSL加密的，允许，否则拒绝
     * user: UserFilter，如果是已知用户，允许访问，判别方法：principal是否被标记，或者说，任何已验证的，已被“记住”的均可访问
     */
    @Bean
    public ShiroFilterChainDefinition shiroFilterChainDefinition() {
        DefaultShiroFilterChainDefinition shiroFilterChainDefinition = new DefaultShiroFilterChainDefinition();
        shiroFilterChainDefinition.addPathDefinition("/doLogin", "anon");
        shiroFilterChainDefinition.addPathDefinition("/**", "authc");
        return shiroFilterChainDefinition;
    }

    /**
     * 设置SecurityManager
     */
    @PostConstruct
    public void init() {

        SecurityUtils.setSecurityManager(securityManager());
    }

    /**
     * 关于SecurityUtils.getSubject()的个人理解，因为tomcat是多线程的，所以每次获取的Subject是怎么保证是当前与应用交互的Subject的呢？
     * 答案就是ThreadLocal。这个类利用Map针对每个线程获取线程的独立于其他线程的域，怎么做到的呢？
     * 这个Map的key不是不同的，而是全部是ThreadLocal类，但是有多个Map，每个线程都保存了一个Map，每个Map都有一个key为ThreadLocal的键值对。
     * 这个和我们平时使用的Map有点不一样，平时想保存多个键值对，无非多个键对应多个值，这个是多个Map对应多个值，每个Map都有某一确定的值
     * 想获取不同的值不是通过改变key的方式而是，通过改变Map的方式来实现，而Map绑定在线程上，遂直接获取当前线程的Map就好。
     * 所以SecurityUtils.getSubject()是获取当前线程的Map的key为"ThreadContext_SUBJECT_KEY"的键值对的值来获取Subject的。
     * @return NA
     */
    @Bean
    public DefaultWebSecurityManager securityManager() {

        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setSessionManager(sessionManager());
        securityManager.setCacheManager(cacheManager());
        securityManager.setRealm(webRealm);
        return securityManager;
    }

    /**
     * 设置会话管理器
     */
    @Bean
    public SessionManager sessionManager() {

        DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
        return sessionManager;
    }

    /**
     * 设置缓存管理器
     */
    @Bean
    public CacheManager cacheManager() {

        EhCacheManager ehCacheManager = new EhCacheManager();
        return ehCacheManager;
    }

    /**
     * 开启注解声明，不然会导致注解方法没法使用
     */
    @Bean
    @ConditionalOnMissingBean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        return new DefaultAdvisorAutoProxyCreator();
    }

}

```
### 登录处理以及未验证处理Controller：
``` JAVA
package com.learn.shiro.controller;

import com.learn.shiro.result.ResultBean;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.subject.Subject;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author SuanCaiYv
 * @time 2020/6/9 下午9:12
 */
@RestController
public class ControllerOne {

    /**
     * 实现真正的登录功能
     */
    @RequestMapping("/doLogin")
    public ResultBean<Void> doLogin(String email, String password) {
        UsernamePasswordToken token = new UsernamePasswordToken();
        token.setUsername(email);
        token.setPassword(password.toCharArray());
        Subject subject = SecurityUtils.getSubject();
        ResultBean<Void> resultBean = new ResultBean<>();
        // 如果已有用户登录
        if (subject.isAuthenticated()) {
            return resultBean;
        }
        try {
            subject.login(token);
            token.setRememberMe(true);
            resultBean.setMsg("登录成功");
            resultBean.setCode(ResultBean.ALL_PASSED);
        } catch (IncorrectCredentialsException e) {
            resultBean.setMsg("密码错误");
            resultBean.setCode(ResultBean.INCORRECT_PASSWORD);
        } catch (ExpiredCredentialsException e) {
            resultBean.setMsg("密码过期");
            resultBean.setCode(ResultBean.LOCKED_USER);
        } catch (UnknownAccountException e) {
            resultBean.setMsg("用户未注册");
            resultBean.setCode(ResultBean.UNSIGNED_USER);
        }
        return resultBean;
    }

    /**
     * 登出设置
     */
    @RequestMapping("/logout")
    public ResultBean<Void> logout() {
        Subject subject = SecurityUtils.getSubject();
        subject.logout();
        return new ResultBean<>();
    }

    /**
     * 设置未验证的映射路径
     */
    @RequestMapping("/unauth")
    public String error() {
        return "unauth";
    }

    /**
     * 设置未登录时的跳转路径
     */
    @RequestMapping("/login")
    public String  login() {
        return "please login!";
    }
}

```
### 权限测试Controller：
``` JAVA
package com.learn.shiro.controller;

import com.learn.shiro.result.ResultBean;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authz.annotation.Logical;
import org.apache.shiro.authz.annotation.RequiresAuthentication;
import org.apache.shiro.authz.annotation.RequiresRoles;
import org.apache.shiro.authz.permission.WildcardPermission;
import org.apache.shiro.subject.Subject;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author SuanCaiYv
 * @time 2020/6/10 下午8:43
 */
@RestController
public class ControllerTwo {

    /**
     * 要求操作者必须拥有角色身份，同时必须是已验证的，否则抛异常
     */
    @RequiresAuthentication
    @RequiresRoles(value = {"com", "adm", "hed"}, logical = Logical.OR)
    @RequestMapping("/task/lunch")
    public ResultBean<Void> f1() {
        Subject subject = SecurityUtils.getSubject();
        ResultBean<Void> resultBean = new ResultBean<>();
        if (subject.isPermitted(new WildcardPermission("*:lun_tsk"))) {
            resultBean.setCode(ResultBean.ALL_PASSED);
            resultBean.setMsg("发布成功");
        }
        else {
            resultBean.setCode(ResultBean.ACCESS_DENIED);
            resultBean.setMsg("您貌似没有权限");
        }
        return resultBean;
    }
}

```
### 验证与授权实现类：
``` JAVA
package com.learn.shiro.dao.shiro;

import com.learn.shiro.pojo.shiro.SysUser;
import org.apache.shiro.authc.*;
import org.apache.shiro.authc.credential.HashedCredentialsMatcher;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.Permission;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.authz.permission.WildcardPermission;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.util.ByteSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * @author SuanCaiYv
 * @time 2020/6/10 下午11:22
 */
@Component
public class WebRealm extends AuthorizingRealm {

    private static final HashedCredentialsMatcher credentialsMatcher;

    private final SysUserMapper sysUserMapper;

    private final SysUserRolesMapper sysUserRolesMapper;

    private final SysUserPermissionsMapper sysUserPermissionsMapper;

    private final SysRoleMapper sysRoleMapper;

    private final SysPermissionMapper sysPermissionMapper;

    private final SysRolePermissionsMapper sysRolePermissionsMapper;


    /**
     * 这里的设置和你的加密方式一致
     */
    static {
        credentialsMatcher = new HashedCredentialsMatcher();
        credentialsMatcher.setHashAlgorithmName("MD5");
        credentialsMatcher.setHashIterations(1024);
        credentialsMatcher.setStoredCredentialsHexEncoded(true);
    }

    /**
     * 这里一定要设置setCredentialsMatcher()，不然会造成密码匹配失败
     */
    public WebRealm(@Autowired SysUserMapper sysUserMapper, @Autowired SysUserRolesMapper sysUserRolesMapper,
                    @Autowired SysUserPermissionsMapper sysUserPermissionsMapper, @Autowired SysRoleMapper sysRoleMapper,
                    @Autowired SysPermissionMapper sysPermissionMapper, @Autowired SysRolePermissionsMapper sysRolePermissionsMapper) {
        this.setCredentialsMatcher(credentialsMatcher);
        this.sysUserMapper = sysUserMapper;
        this.sysUserRolesMapper = sysUserRolesMapper;
        this.sysUserPermissionsMapper = sysUserPermissionsMapper;
        this.sysRoleMapper = sysRoleMapper;
        this.sysPermissionMapper = sysPermissionMapper;
        this.sysRolePermissionsMapper = sysRolePermissionsMapper;
    }


    /**
     * 授权
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        String username = (String) principals.getPrimaryPrincipal();
        SysUser sysUser = sysUserMapper.select(username);
        List<Long> roleIds = sysUserRolesMapper.selectByUser(sysUser.getId());
        List<Long> permissionIds = sysUserPermissionsMapper.selectByUser(sysUser.getId());
        Set<String> roles = new HashSet<>();
        Set<Permission> permissions = new HashSet<>();
        // 获取用户权限
        for (Long roleId : roleIds) {
            roles.add(sysRoleMapper.selectById(roleId).getName());
            for (Long permissionId : permissionIds) {
                WildcardPermission permission = new WildcardPermission(sysRoleMapper.selectById(roleId).getName()+
                        ":"+sysPermissionMapper.selectById(permissionId).getName());
                permissions.add(permission);
            }
        }
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        // 添加权限的过程
        simpleAuthorizationInfo.setRoles(roles);
        simpleAuthorizationInfo.setObjectPermissions(permissions);
        return simpleAuthorizationInfo;
    }

    /**
     * 验证
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        String username = (String) token.getPrincipal();
        SysUser sysUser = sysUserMapper.select(username);
        if (sysUser == null) {
            throw new UnknownAccountException("Unsigned User!");
        }
        // 验证用户身份
        SimpleAuthenticationInfo simpleAuthenticationInfo = new SimpleAuthenticationInfo(token.getPrincipal(), sysUser.getPassword(), ByteSource.Util.bytes(sysUser.getSalt()), this.getName());
        return simpleAuthenticationInfo;
    }
}

```
### 配置文件：
``` YML
server:
  port: 8190
shiro:
  sessionManager:
    sessionIdCookieEnabled: true
    sessionIdUrlRewritingEnabled: true
  web:
    enabled: true
  successUrl: /index
  loginUrl: /login
  unauthorizedUrl: /unauth
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/shiro_test
    username: root
    password: root
mybatis:
  config-location: classpath:mybatis-config.xml
  mapper-locations: classpath:mappers/*.xml

```
### 最后贴上添加用户数据的操作：
``` JAVA
package com.learn.shiro;

import com.learn.shiro.dao.shiro.SysPermissionMapper;
import com.learn.shiro.dao.shiro.SysRoleMapper;
import com.learn.shiro.dao.shiro.SysUserMapper;
import com.learn.shiro.pojo.shiro.SysPermission;
import com.learn.shiro.pojo.shiro.SysUser;
import com.learn.shiro.util.BaseUtil;
import org.apache.shiro.crypto.hash.SimpleHash;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class ShiroApplicationTests {

    @Autowired
    private SysUserMapper sysUserMapper;

    @Autowired
    private SysRoleMapper sysRoleMapper;

    @Autowired
    private SysPermissionMapper sysPermissionMapper;

    @Test
    void contextLoads() {

        SysUser sysUser = new SysUser();
        sysUser.setUsername("2021601470@qq.com");
        sysUser.setNickname("水煮鱼");
        sysUser.setState(0);
        String salt = BaseUtil.getSalt();
        sysUser.setSalt(salt);
        SimpleHash simpleHash = new SimpleHash("MD5", "123456", salt, 1024);
        sysUser.setPassword(simpleHash.toHex());
        sysUserMapper.insert(sysUser);
    }
}

```
至此，一个使用Shiro的SpringBoot应用就设置完成了！

来看看运行结果吧！



![](https://user-gold-cdn.xitu.io/2020/6/15/172b75e6492eed3d?w=2057&h=1193&f=png&s=169866)
![](https://user-gold-cdn.xitu.io/2020/6/15/172b75e6f359008f?w=2056&h=1194&f=png&s=167939)
![](https://user-gold-cdn.xitu.io/2020/6/15/172b75e793a7ba5a?w=2056&h=1193&f=png&s=161738)
![](https://user-gold-cdn.xitu.io/2020/6/15/172b75e83f686650?w=2057&h=1191&f=png&s=172754)
![](https://user-gold-cdn.xitu.io/2020/6/15/172b75e8d53fa4fd?w=2055&h=1194&f=png&s=172295)

完整代码地址：https://github.com/SuanCaiYv/shiro/tree/springbootshiro