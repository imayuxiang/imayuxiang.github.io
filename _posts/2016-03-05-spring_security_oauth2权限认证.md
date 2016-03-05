---
layout: post
title: spring security oauth2 实现权限认证
category : misc
tags : [java web]
---

spring security oauth 整合

oauth2的官方文档不容易看懂，spring-security-oauth2的example也不太容易懂
http://blog.csdn.net/monkeyking1987/article/details/16828059
http://www.cnblogs.com/smarterplanet/p/4088479.html
http://porterhead.blogspot.co.id/2014/05/securing-rest-services-with-spring.html
这几篇文档写的比较清楚，仔细跟着走一遍看看代码就能学明白

跟着自己配一遍

配置完之后可实现三种方式的资源获取
第一种 普通的web端认证 通过用户名密码 提交到login.do（spring security内部实现了login.do的验证获取方式）用户名和密码的参数为j_username, j_password

第二种 通过用户名、密码、grant_type、client_id、client_secret提交到/oauth/token获得access_token，之后带着access_token去获得资源

第三种 通过client_id提交到oauth/authorize获得code，再通过code获得access_token 
	"网站"提供了一个超链接，名字叫"Login using appDemo"，链接为：
	http://localhost:8080/appDemo/oauth/authorize?client_id=m1&redirect_uri=http%3a	%2f%2flocalhost%3a8080%2f&response_type=code&scope=read
	
先跳转到登录界面让用户登录 跳转回了刚才的returnUri，并且添加了code参数：http://localhost:8080/?code=VOUIRc 
现在，client拿到了code，这个code就是Authorization Code，下面这步是client要做的
由于没有现成的client,我们可以用一些REST tool来模拟client向appDemo申请AccessToken的过程
tool包括：firefox的REST client，Chrome的POSTMan，真正使用的时候，client需要使用后台代码发http请求，前端Ajax会有跨域限制

	Client可以直接发起个请求(Method要用Post)来获取Access Token:
	http://localhost:8080/appDemo/oauth/token?code=g6hW13&client_id=m1&client_secret=s1&grant_type=authorization_code&redirect_uri=http%3a%2f%2flocalhost%3a8080%2f

注意把这个URL中的code替换成前面一步生成的code
	
配置文件如下

针对不同resource的http配置, 由于上面配置了两个resource

    <http pattern="/app/**" create-session="never"
          entry-point-ref="oauth2AuthenticationEntryPoint"
          access-decision-manager-ref="oauth2AccessDecisionManager">
        <anonymous enabled="false"/>
        <intercept-url pattern="/app/**" access="ROLE_USER,SCOPE_READ"/>
        <custom-filter ref="mobileResourceServer" before="PRE_AUTH_FILTER"/>
        <access-denied-handler ref="oauthAccessDeniedHandler"/>
    </http>

    <!--<oauth2:resource-server id="unityResourceServer"-->
                            <!--resource-id="unity-resource" token-services-ref="tokenServices"/>-->
    <oauth2:resource-server id="mobileResourceServer"
                            resource-id="mobile-resource" token-services-ref="tokenServices"/>

 /oauth/token 的http 配置, 用于监听该URL的请求, 核心
 
    <http pattern="/oauth/token" create-session="stateless"
          authentication-manager-ref="oauth2AuthenticationManager"
          entry-point-ref="oauth2AuthenticationEntryPoint">
        <intercept-url pattern="/oauth/token" access="IS_AUTHENTICATED_FULLY"/>
        <anonymous enabled="false"/>
        <http-basic entry-point-ref="oauth2AuthenticationEntryPoint"/>
        <custom-filter ref="clientCredentialsTokenEndpointFilter"
                       before="BASIC_AUTH_FILTER"/>
        <access-denied-handler error-page="/login?authorization_error=2"/>
    </http>

oauth2 AuthenticationManager配置; 在整个配置中,有两个AuthenticationManager需要配置

    <authentication-manager id="oauth2AuthenticationManager">
        <authentication-provider user-service-ref="oauth2ClientDetailsUserService"/>
    </authentication-manager>

第二个AuthenticationManager用于向获取UserDetails信息

    <beans:bean id="myUserDetailsService" class="com.july.aboutiming.security.MyUserDetailsService"/>
    <authentication-manager alias="authenticationManager">
        <authentication-provider user-service-ref="myUserDetailsService">
            <password-encoder hash="md5"/>
        </authentication-provider>
    </authentication-manager>

  ClientDetailsUserDetailsService配置, 该类实现了Spring security中 UserDetailsService 接口
  
    <beans:bean id="oauth2ClientDetailsUserService"
                class="org.springframework.security.oauth2.provider.client.ClientDetailsUserDetailsService">
        <beans:constructor-arg ref="clientDetailsService"/>
    </beans:bean>

  ClientCredentialsTokenEndpointFilter配置, 该Filter将作用于Spring Security的chain 链条中
  
    <beans:bean id="clientCredentialsTokenEndpointFilter"
                class="org.springframework.security.oauth2.provider.client.ClientCredentialsTokenEndpointFilter">
        <beans:property name="authenticationManager" ref="oauth2AuthenticationManager"/>
    </beans:bean>

  OAuth2AuthenticationEntryPoint配置
    
    <beans:bean id="oauth2AuthenticationEntryPoint"
                class="org.springframework.security.oauth2.provider.error.OAuth2AuthenticationEntryPoint"/>
 
 OAuth2AccessDeniedHandler配置, 实现AccessDeniedHandler接口
 
    <beans:bean id="oauthAccessDeniedHandler"
                class="org.springframework.security.oauth2.provider.error.OAuth2AccessDeniedHandler"/>
    <beans:bean id="oauthUserApprovalHandler"
                class="org.springframework.security.oauth2.provider.approval.DefaultUserApprovalHandler"/>

 ClientDetailsService 配置, 使用JdbcClientDetailsService,
     这里直接用内存的 也需要提供dataSource, 替换demo中直接配置在配置文件中

    <oauth2:client-details-service id="clientDetailsService">

        <!-- Allow access to test clients -->
        <oauth2:client
                client-id="353b302c44574f565045687e534e7d6a"
                secret="286924697e615a672a646a493545646c"
                authorized-grant-types="password,authorization_code,refresh_token,implicit"
                resource-ids="pic-resource"
                authorities="ROLE_ADMIN"
                access-token-validity="5184000"
                refresh-token-validity="5184000"
                scope="read, write, trust"
        />

        <!-- Web Application clients -->
        <oauth2:client
                client-id="7b5a38705d7b3562655925406a652e32"
                secret="655f523128212d6e70634446224c2a48"
                authorized-grant-types="password,authorization_code,refresh_token,implicit"
                resource-ids="pic-resource"
                scope="read, write, trust"
                authorities="ROLE_WEB"
                access-token-validity="5184000"
                refresh-token-validity="5184000"
        />

        <!-- iOS clients -->
        <oauth2:client
                client-id="5e572e694e4d61763b567059273a4d3d"
                secret="316457735c4055642744596b302e2151"
                authorized-grant-types="password,authorization_code,refresh_token,implicit"
                resource-ids="pic-resource"
                scope="read, write, trust"
                authorities="ROLE_IOS"
                access-token-validity="5184000"
                refresh-token-validity="5184000"
        />

        <!-- Android clients -->
        <oauth2:client
                client-id="302a7d556175264c7e5b326827497349"
                secret="4770414c283a20347c7b553650425773"
                authorized-grant-types="password,authorization_code,refresh_token,implicit"
                resource-ids="pic-resource"
                scope="read, write, trust"
                authorities="ROLE_ANDROID"
                access-token-validity="5184000"
                refresh-token-validity="5184000"
        />

    </oauth2:client-details-service>

 TokenStore, 使用JdbcTokenStore, 将token信息存放数据库, 需要提供一个dataSource对象; 也可使用InMemoryTokenStore存于内存中
 
    <beans:bean id="tokenStore"
                class="org.springframework.security.oauth2.provider.token.store.InMemoryTokenStore">
    </beans:bean>
    <!--数据库存储方式-->
    <!--<beans:bean id="tokenStore" class="org.springframework.security.oauth2.provider.token.JdbcTokenStore">-->
    <!--<beans:constructor-arg index="0" ref="dataSource"/>-->
    <!--</beans:bean>-->
    
 TokenServices; 需要注入TokenStore,如果允许刷新token 请将supportRefreshToken 的值设置为true, 默认为不允许
 
    <beans:bean id="tokenServices"
                class="org.springframework.security.oauth2.provider.token.DefaultTokenServices">
        <beans:property name="tokenStore" ref="tokenStore"/>
        <beans:property name="supportRefreshToken" value="true"/>
        <beans:property name="clientDetailsService" ref="clientDetailsService"/>
    </beans:bean>

 authorization-server配置, 核心  这里可以配置授权方式 4种 implicit还没弄明白
 
    <oauth2:authorization-server
            client-details-service-ref="clientDetailsService" token-services-ref="tokenServices"
            user-approval-handler-ref="oauthUserApprovalHandler"
            user-approval-page="oauth_approval" error-page="oauth_error">
        <oauth2:authorization-code/>
        <oauth2:implicit/>
        <oauth2:refresh-token/>
        <oauth2:client-credentials/>
        <oauth2:password/>
    </oauth2:authorization-server>

  Oauth2 AccessDecisionManager配置, 这儿在默认的Spring Security AccessDecisionManager的基础上添加了ScopeVoter
  
    <beans:bean id="oauth2AccessDecisionManager"
                class="org.springframework.security.access.vote.UnanimousBased">
        <beans:constructor-arg>
            <beans:list>
                <beans:bean
                        class="org.springframework.security.oauth2.provider.vote.ScopeVoter"/>
                <beans:bean class="org.springframework.security.access.vote.RoleVoter"/>
                <beans:bean
                        class="org.springframework.security.access.vote.AuthenticatedVoter"/>
            </beans:list>
        </beans:constructor-arg>
    </beans:bean>

默认的http配置,给/oauth/** 设置权限

    <http  authentication-manager-ref="authenticationManager" disable-url-rewriting="true"
           access-denied-page="/login?authorization_error=2">
        <intercept-url pattern="/backstage/**" access="ROLE_ADMIN"/>
        <intercept-url pattern="/healthRecord/**" access="ROLE_USER"/>
        <intercept-url pattern="/oauth/**" access="ROLE_USER"/>
        <intercept-url pattern="/**" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
        <form-login authentication-failure-url="/login?authentication_error=1" default-target-url="/"
                    login-page="/login" login-processing-url="/login.do"/>
        <logout logout-success-url="/" logout-url="/logout.do"/>
        <!--<custom-filter ref="unityResourceServer" before="PRE_AUTH_FILTER"/>-->
        <anonymous/>
    </http>
        
security相对比较复杂，spring跟微软的风格差不多，什么都弄到一起，demo又写的一般不花点时间看不太明白，一时半会很难弄明白，相对shiro比较简单易懂，容易上手，操作起来也更简单，等有时间了再弄一个shiro的