---
layout: post
title: "Spring Cloud 项目国际化"
subtitle: 'Spring Cloud i18n demo'
author: "yqsas"
header-style: text
catalog: true
tags:
  - i18n
  - Spring Cloud
---

## Spring Cloud 项目国际化

### 0. 前言

最近几天给项目做了国际化相关工作，以此记录。

Spring Boot 默认就支持国际化，而且不需要我们过多的做什么配置，只需要在 resources/下定义国际化配置文件即可，注意名称必须以 messages 开头。
但是考虑到配置管理以及自定义解析需求，一般还是另外自定义配置类。

原理是读取国际化配置文件，以 **编码=内容** 为映射记录。当收到一个请求时，使用配置的 LocaleResolver 解析当前请求语言，
最后使用 MessageSource 获取编码映射的实际语言内容。

Demo 代码 github 地址：[https://github.com/yqsas/spring-cloud-i18n-demo](https://github.com/yqsas/spring-cloud-i18n-demo)

### 1. Spring 国际化配置

```yaml
spring:
  messages:
    encoding: UTF-8
    basename: i18n/messages
```

### 2. 国际化语言配置文件

在 resources 下创建 i18n 目录，并在其中新建文件：messages.properties、messages_en_US.properties、messages_zh_CN.properties。
，注意以 messages 开头。

配置文件中填写 **编码=内容** 形式的映射记录。

如：
messages_en_US.properties

```properties
user.welcome=Welcome
```

messages_zh_CN.properties

```properties
user.welcome=欢迎
```

### 3. 自定义语言区域解析器

Spring 在 org.springframework.web.servlet.i18n 包下，提供默认几种语言区域解析器：CookieLocaleResolver、AcceptHeaderLocaleResolver、
FixedLocaleResolver、SessionLocaleResolver 等，几种方式顾名思义，各有用途。

但是在 Spring Cloud 体系中，我们用到 OAuth2 作为鉴权机制，后端无 Session 可用，所以首先排除 SessionLocaleResolver；
其次微服务之间接口调用使用的 Feign，无法传递 Cookie，所以也排除 CookieLocaleResolver；
并且 AcceptHeaderLocaleResolver 使用 Header 中的 **"Accept-Language"** 属性来判断客户端语言环境，排除。

最终确定自定义语言区域解析器，使用请求头中自定义属性作为解析依据。

```java
public class LocaleHeaderLocaleResolver implements LocaleContextResolver {

    public static final String LOCALE_REQUEST_ATTRIBUTE_NAME = LocaleHeaderLocaleResolver.class.getName() + ".LOCALE";

    public static final String TIME_ZONE_REQUEST_ATTRIBUTE_NAME = LocaleHeaderLocaleResolver.class.getName() + ".TIME_ZONE";

    @Nullable
    private Locale defaultLocale;
    @Nullable
    private TimeZone defaultTimeZone;

    @Nullable
    private String localeHeadName;

    public void setLocaleHeadName(@Nullable String localeHeadName) {
        this.localeHeadName = localeHeadName;
    }

    @Nullable
    public String getLocaleHeadName() {
        return this.localeHeadName;
    }

    public void setDefaultLocale(@Nullable Locale defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    @Nullable
    public Locale getDefaultLocale() {
        return this.defaultLocale;
    }

    public void setDefaultTimeZone(@Nullable TimeZone defaultTimeZone) {
        this.defaultTimeZone = defaultTimeZone;
    }

    @Nullable
    protected TimeZone getDefaultTimeZone() {
        return this.defaultTimeZone;
    }

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        parseLocaleHeaderIfNecessary(request);
        return (Locale) request.getAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME);
    }

    @Override
    public LocaleContext resolveLocaleContext(final HttpServletRequest request) {
        parseLocaleHeaderIfNecessary(request);
        return new TimeZoneAwareLocaleContext() {
            @Override
            @Nullable
            public Locale getLocale() {
                return (Locale) request.getAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME);
            }

            @Override
            @Nullable
            public TimeZone getTimeZone() {
                return (TimeZone) request.getAttribute(TIME_ZONE_REQUEST_ATTRIBUTE_NAME);
            }
        };
    }

    @Override
    public void setLocaleContext(HttpServletRequest request, @Nullable HttpServletResponse response,
                                 @Nullable LocaleContext localeContext) {
        Assert.notNull(response, "HttpServletResponse is required for LocaleHeaderLocaleResolver");

        Locale locale = null;
        TimeZone timeZone = null;
        if (localeContext != null) {
            locale = localeContext.getLocale();
            if (localeContext instanceof TimeZoneAwareLocaleContext) {
                timeZone = ((TimeZoneAwareLocaleContext) localeContext).getTimeZone();
            }
            response.setHeader(getLocaleHeadName(),
                (locale != null ? locale.toString() : "-") + (timeZone != null ? ' ' + timeZone.getID() : ""));
        }
        request.setAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME,
            (locale != null ? locale : determineDefaultLocale(request)));
        request.setAttribute(TIME_ZONE_REQUEST_ATTRIBUTE_NAME,
            (timeZone != null ? timeZone : determineDefaultTimeZone(request)));
    }

    private void parseLocaleHeaderIfNecessary(HttpServletRequest request) {
        if (request.getAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME) == null) {
            Locale locale = null;
            TimeZone timeZone = null;

            String headName = getLocaleHeadName();
            if (headName != null) {
                String localeStr = request.getHeader(headName);
                if (localeStr != null) {
                    String localePart = localeStr;
                    String timeZonePart = null;
                    int spaceIndex = localePart.indexOf(' ');
                    if (spaceIndex != -1) {
                        localePart = localeStr.substring(0, spaceIndex);
                        timeZonePart = localeStr.substring(spaceIndex + 1);
                    }
                    try {
                        locale = (!"-".equals(localePart) ? parseLocaleValue(localePart) : null);
                        if (timeZonePart != null) {
                            timeZone = StringUtils.parseTimeZoneString(timeZonePart);
                        }
                    } catch (IllegalArgumentException ex) {
                        if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
                            // Error dispatch: ignore locale/timezone parse exceptions
                        } else {
                            throw new IllegalStateException("Invalid locale Header '" + getLocaleHeadName() +
                                "' with value [" + localeStr + "]: " + ex.getMessage());
                        }
                    }
                }
            }

            request.setAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME,
                (locale != null ? locale : determineDefaultLocale(request)));
            request.setAttribute(TIME_ZONE_REQUEST_ATTRIBUTE_NAME,
                (timeZone != null ? timeZone : determineDefaultTimeZone(request)));
        }
    }

    @Nullable
    protected Locale determineDefaultLocale(HttpServletRequest request) {
        Locale defaultLocale = getDefaultLocale();
        if (defaultLocale == null) {
            defaultLocale = request.getLocale();
        }
        return defaultLocale;
    }

    @Nullable
    protected TimeZone determineDefaultTimeZone(HttpServletRequest request) {
        return getDefaultTimeZone();
    }

    @Nullable
    protected Locale parseLocaleValue(String locale) {
        return StringUtils.parseLocaleString(locale);
    }

    @Override
    public void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale) {
        setLocaleContext(request, response, (locale != null ? new SimpleLocaleContext(locale) : null));
    }

}
```

### 4. 国际化配置类

使用自定义区域解析类。

```java
@Configuration
public class LocaleConfig {

    private final String LOCAL_HEAD_NAME = "Locale";

    @Bean
    public LocaleResolver localeResolver() {
        LocaleHeaderLocaleResolver localeResolver = new LocaleHeaderLocaleResolver();
        localeResolver.setLocaleHeadName(LOCAL_HEAD_NAME);
        localeResolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        return localeResolver;
    }

    @Bean
    public WebMvcConfigurer localeInterceptor() {
        return new WebMvcConfigurer() {
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                LocaleChangeInterceptor localeInterceptor = new LocaleChangeInterceptor();
                localeInterceptor.setParamName(LOCAL_HEAD_NAME);
                registry.addInterceptor(localeInterceptor);
            }
        };
    }
}
```

### 5. 其他

- 配合 Feign 使用，需要 Feign 配置转发请求 Header；

- 工具类提供静态方法获取当前请求线程时区，以及编码内容映射转换；

- 前端配合使用用户信息中的 lang 属性，设置当前用户使用的语言，在所有请求头中使用我们自定义属性 **"Locale"** 与后端交互。
