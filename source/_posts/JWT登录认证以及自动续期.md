---
title: JWT登录认证以及自动续期
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-24 14:43:48
password:
summary:
tags:
	- jwt
	- 自动续期
	- java
	- oauth
categories: JWT
---

## 技术选型

要实现认证功能，很容易就会想到 JWT 或者 session，但是两者有啥区别？各自的优缺点？应该 Pick 谁？夺命三连

## 区别

基于 session 和基于 JWT 的方式的主要区别就是用户的状态保存的位置，session 是保存在服务端的，而 JWT 是保存在客户端的

## 认证流程

基于 session 的认证流程

- 用户在浏览器中输入用户名和密码，服务器通过密码校验后生成一个 session 并保存到数据库
- 服务器为用户生成一个 sessionId，并将具有 sesssionId 的 cookie 放置在用户浏览器中，在后续的请求中都将带有这个 cookie 信息进行访问
- 服务器获取 cookie，通过获取 cookie 中的 sessionId 查找数据库判断当前请求是否有效

基于 JWT 的认证流程

- 用户在浏览器中输入用户名和密码，服务器通过密码校验后生成一个 token 并保存到数据库
- 前端获取到 token，存储到 cookie 或者 local storage 中，在后续的请求中都将带有这个 token 信息进行访问
- 服务器获取 token 值，通过查找数据库判断当前 token 是否有效

## 优缺点

JWT 保存在客户端，在分布式环境下不需要做额外工作。而 session 因为保存在服务端，分布式环境下需要实现多机数据共享 session 一般需要结合 Cookie 实现认证，所以需要浏览器支持 cookie，因此移动端无法使用 session 认证方案

## 安全性

JWT 的 payload 使用的是 base64 编码的，因此在 JWT 中不能存储敏感数据。而 session 的信息是存在服务端的，相对来说更安全

如果在 JWT 中存储了敏感信息，可以解码出来非常的不安全

## 性能

经过编码之后 JWT 将非常长，cookie 的限制大小一般是 4k，cookie 很可能放不下，所以 JWT 一般放在 local storage 里面。并且用户在系统中的每一次 http 请求都会把 JWT 携带在 Header 里面，HTTP 请求的 Header 可能比 Body 还要大。而 sessionId 只是很短的一个字符串，因此使用 JWT 的 HTTP 请求比使用 session 的开销大得多

## 一次性

无状态是 JWT 的特点，但也导致了这个问题，JWT 是一次性的。想修改里面的内容，就必须签发一个新的 JWT

## 无法废弃

一旦签发一个 JWT，在到期之前就会始终有效，无法中途废弃。若想废弃，一种常用的处理手段是结合 redis

## 续签

如果使用 JWT 做会话管理，传统的 cookie 续签方案一般都是框架自带的，session 有效期 30 分钟，30 分钟内如果有访问，有效期被刷新至 30 分钟。一样的道理，要改变 JWT 的有效时间，就要签发新的 JWT。

最简单的一种方式是每次请求刷新 JWT，即每个 HTTP 请求都返回一个新的 JWT。这个方法不仅暴力不优雅，而且每次请求都要做 JWT 的加密解密，会带来性能问题。另一种方法是在 redis 中单独为每个 JWT 设置过期时间，每次访问时刷新 JWT 的过期时间

## 选择 JWT 或 session

我投 JWT 一票，JWT 有很多缺点，但是在分布式环境下不需要像 session 一样额外实现多机数据共享，虽然 seesion 的多机数据共享可以通过粘性 session、session 共享、session 复制、持久化 session、terracoa 实现 seesion 复制等多种成熟的方案来解决这个问题。但是 JWT 不需要额外的工作，使用 JWT 不香吗？且 JWT 一次性的缺点可以结合 redis 进行弥补。

## 功能实现

### JWT 所需依赖

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.10.3</version>
</dependency>

```

### JWT 工具类

```java
public class JWTUtil {
    private static final Logger logger = LoggerFactory.getLogger(JWTUtil.class);

    //私钥
    private static final String TOKEN_SECRET = "123456";

    /**
     * 生成token，自定义过期时间 毫秒
     *
     * @param userTokenDTO
     * @return
     */
    public static String generateToken(UserTokenDTO userTokenDTO) {
        try {
            // 私钥和加密算法
            Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);
            // 设置头部信息
            Map<String, Object> header = new HashMap<>(2);
            header.put("Type", "Jwt");
            header.put("alg", "HS256");

            return JWT.create()
                    .withHeader(header)
                    .withClaim("token", JSONObject.toJSONString(userTokenDTO))
                    //.withExpiresAt(date)
                    .sign(algorithm);
        } catch (Exception e) {
            logger.error("generate token occur error, error is:{}", e);
            return null;
        }
    }

    /**
     * 检验token是否正确
     *
     * @param token
     * @return
     */
    public static UserTokenDTO parseToken(String token) {
        Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);
        JWTVerifier verifier = JWT.require(algorithm).build();
        DecodedJWT jwt = verifier.verify(token);
        String tokenInfo = jwt.getClaim("token").asString();
        return JSON.parseObject(tokenInfo, UserTokenDTO.class);
    }
}
```

#### 说明：

- 生成的 token 中不带有过期时间，token 的过期时间由 redis 进行管理
- UserTokenDTO 中不带有敏感信息，如 password 字段不会出现在 token 中

### Redis 工具类

```java
public final class RedisServiceImpl implements RedisService {
    /**
     * 过期时长
     */
    private final Long DURATION = 1 * 24 * 60 * 60 * 1000L;

    @Resource
    private RedisTemplate redisTemplate;

    private ValueOperations<String, String> valueOperations;

    @PostConstruct
    public void init() {
        RedisSerializer redisSerializer = new StringRedisSerializer();
        redisTemplate.setKeySerializer(redisSerializer);
        redisTemplate.setValueSerializer(redisSerializer);
        redisTemplate.setHashKeySerializer(redisSerializer);
        redisTemplate.setHashValueSerializer(redisSerializer);
        valueOperations = redisTemplate.opsForValue();
    }

    @Override
    public void set(String key, String value) {
        valueOperations.set(key, value, DURATION, TimeUnit.MILLISECONDS);
        log.info("key={}, value is: {} into redis cache", key, value);
    }

    @Override
    public String get(String key) {
        String redisValue = valueOperations.get(key);
        log.info("get from redis, value is: {}", redisValue);
        return redisValue;
    }

    @Override
    public boolean delete(String key) {
        boolean result = redisTemplate.delete(key);
        log.info("delete from redis, key is: {}", key);
        return result;
    }

    @Override
    public Long getExpireTime(String key) {
        return valueOperations.getOperations().getExpire(key);
    }
}
```

RedisTemplate 简单封装

## 业务实现

### 登陆功能

```java
public String login(LoginUserVO loginUserVO) {
    //1.判断用户名密码是否正确
    UserPO userPO = userMapper.getByUsername(loginUserVO.getUsername());
    if (userPO == null) {
        throw new UserException(ErrorCodeEnum.TNP1001001);
    }
    if (!loginUserVO.getPassword().equals(userPO.getPassword())) {
        throw new UserException(ErrorCodeEnum.TNP1001002);
    }

    //2.用户名密码正确生成token
    UserTokenDTO userTokenDTO = new UserTokenDTO();
    PropertiesUtil.copyProperties(userTokenDTO, loginUserVO);
    userTokenDTO.setId(userPO.getId());
    userTokenDTO.setGmtCreate(System.currentTimeMillis());
    String token = JWTUtil.generateToken(userTokenDTO);

    //3.存入token至redis
    redisService.set(userPO.getId(), token);
    return token;
}
```

#### 说明：

- 判断用户名密码是否正确
- 用户名密码正确则生成 token
- 将生成的 token 保存至 redis

### 登出功能

```java
public boolean loginOut(String id) {
     boolean result = redisService.delete(id);
     if (!redisService.delete(id)) {
        throw new UserException(ErrorCodeEnum.TNP1001003);
     }

     return result;
}
```

将对应的 key 删除即可

### 更新密码功能

```java
public String updatePassword(UpdatePasswordUserVO updatePasswordUserVO) {
    //1.修改密码
    UserPO userPO = UserPO.builder().password(updatePasswordUserVO.getPassword())
            .id(updatePasswordUserVO.getId())
            .build();
    UserPO user = userMapper.getById(updatePasswordUserVO.getId());
    if (user == null) {
        throw new UserException(ErrorCodeEnum.TNP1001001);
    }

    if (userMapper.updatePassword(userPO) != 1) {
        throw new UserException(ErrorCodeEnum.TNP1001005);
    }
    //2.生成新的token
    UserTokenDTO userTokenDTO = UserTokenDTO.builder()
            .id(updatePasswordUserVO.getId())
            .username(user.getUsername())
            .gmtCreate(System.currentTimeMillis()).build();
    String token = JWTUtil.generateToken(userTokenDTO);
    //3.更新token
    redisService.set(user.getId(), token);
    return token;
}
```

#### 说明

> 更新用户密码时需要重新生成新的 token，并将新的 token 返回给前端，由前端更新保存在 local storage 中的 token，同时更新存储在 redis 中的 token，这样实现可以避免用户重新登陆，用户体验感不至于太差

#### 其他说明

在实际项目中，用户分为普通用户和管理员用户，只有管理员用户拥有删除用户的权限，这一块功能也是涉及 token 操作的，但是我太懒了，demo 工程就不写了

在实际项目中，密码传输是加密过的

### 拦截器类

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
    String authToken = request.getHeader("Authorization");
    String token = authToken.substring("Bearer".length() + 1).trim();
    UserTokenDTO userTokenDTO = JWTUtil.parseToken(token);
    //1.判断请求是否有效
    if (redisService.get(userTokenDTO.getId()) == null
            || !redisService.get(userTokenDTO.getId()).equals(token)) {
        return false;
    }

    //2.判断是否需要续期
    if (redisService.getExpireTime(userTokenDTO.getId()) < 1 * 60 * 30) {
        redisService.set(userTokenDTO.getId(), token);
        log.error("update token info, id is:{}, user info is:{}", userTokenDTO.getId(), token);
    }
    return true;
}
```

#### 说明：

拦截器中主要做两件事，一是对 token 进行校验，二是判断 token 是否需要进行续期

##### token 校验：

- 判断 id 对应的 token 是否不存在，不存在则 token 过期
- 若 token 存在则比较 token 是否一致，保证同一时间只有一个用户操作

##### token 自动续期：

为了不频繁操作 redis，只有当离过期时间只有 30 分钟时才更新过期时间

### 拦截器配置类

```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authenticateInterceptor())
                .excludePathPatterns("/logout/**")
                .excludePathPatterns("/login/**")
                .addPathPatterns("/**");
    }

    @Bean
    public AuthenticateInterceptor authenticateInterceptor() {
        return new AuthenticateInterceptor();
    }
}
```
