---
layout: post
title: "Spring Security OAuth2 表单登录 refresh token 未生成问题"
subtitle: 'Spring Security OAuth2 formLogin missing refresh token'
author: "yqsas"
header-style: text
catalog: true
tags:
  - Spring
  - Java
---

之前集成 OAuth2 后，一直用默认的登录路径/oauth/token，但是需要把客户端和客户端密钥明文传输，且登录成功后，后端后续处理不方便。于是打算使用自定义 formLogin 路径方式提供登录。

## 过程

1. 配置 AuthorizationServerConfig，使用 redisTokenStore 配置 DefaultTokenServices，使用 JdbcClientDetailsService 配置 ClientDetailsService。
2. 新建 FormAuthenticationConfig，配置 HttpSecurity http 访问 formLogin 方法，并自定义登录路径 loginProcessingUrl 为/oauth/form，再自定义 successHandler 类（继承 SavedRequestAwareAuthenticationSuccessHandler），处理登录成功后手动生成 token 并返回给前端。
3. 问题出现：使用 TokenServices.createAccessToken，手动创建出来的 token 无 refreshToken，而使用/oauth/token 端点则正常。

## 问题定位

1. 追踪 TokenServices.createAccessToken 方法，发现其中生成 refresh_token 的方法，是通过判断 ClientDetail 里的认证方式是否包含 refresh_token 来确定：

```java
protected boolean isSupportRefreshToken(OAuth2Request clientAuth) {
        if (this.clientDetailsService != null) {
            ClientDetails client = this.clientDetailsService.loadClientByClientId(clientAuth.getClientId());
            return client.getAuthorizedGrantTypes().contains("refresh_token");
        } else {
            return this.supportRefreshToken;
        }
    }
```

确认项目中 ClientDetail 中确实包含 refresh_token 后，再断点追踪，才知道 DefaultTokenServices 里面的 clientDetailsService 为空，导致无法获取客户端详情 clientDetails。

## 解决方式

配置 DefaultTokenServices 时

1. 设置 clientDetailsService
2. 设置 supportRefreshToken 为 true（默认 false)，如此所有 token 都需要创建 refreshToken

```java
@Bean
@Primary
public DefaultTokenServices tokenServices() {
    DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
    defaultTokenServices.setTokenStore(redisTokenStore);
    //1
    defaultTokenServices.setClientDetailsService(this.clientDetails());
    //2
    defaultTokenServices.setSupportRefreshToken(true);
    return defaultTokenServices;
}
```

1 和 2 都不可缺少，因为调用刷新 token 方法的时候，DefaultTokenServices 是直接取 supportRefreshToken 值判断，而非调用 isSupportRefreshToken 方法判断，而 supportRefreshToken 默认 false。这算是一个 bug 吧。

## 总结

1. 直接拿开源项目并不十分可靠，需要仔细理解细节；
2. 有 BUG，看源码。
