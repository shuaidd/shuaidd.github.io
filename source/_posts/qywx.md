---
title: 企业微信后端接口客户端-Java版
date: 2021-07-09 13:42:49
tags: 
   - Java 
   - 企业微信 
   - springboot
---
### maven 坐标
```xml
<dependency>
    <groupId>com.github.shuaidd</groupId>
    <artifactId>qywx-spring-boot-api</artifactId>
    <version>2.1.0</version>
</dependency>
```
[源码地址](https://github.com/shuaidd/qywx)

### 使用配置示例
```yaml
qywx:
  corp-id: xxxx（企业号）
  application-list:
  - secret: Kx4sovYN5C0_MEzPY0cOymwbMhGmqdA9VjMFHrAKjdE
    agentId: 1000003
    application-name: little-helper
  - secret: DXB-FXVZNkLGUlLaIJy6CK67WD-dpN1HnPLIzNPo0N4
    agentId: 1000004
    application-name: reporter
  - secret: AfjvAed_ulqhK0OqTprDQ6xOSnqaT34ll2LsRe0D2NA
    application-name: address-book
  url: https://qyapi.weixin.qq.com
  public-path: cgi-bin
```
### 代码示例
```java
/**
* 实例 可以查看 qywx-spring-boot-example 模块
*/
public class WeChatTest extends BaseTest {
    /**
    * 查询用户信息
    */
    @Test
    public void getUser(){
        weChatManager.addressBookService().getUser("13259220281", "address-book");
    }
}
```
