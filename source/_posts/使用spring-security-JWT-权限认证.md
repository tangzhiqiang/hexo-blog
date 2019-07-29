---
title: 使用spring security + JWT 权限认证
date: 2019-07-29 11:18:47
tags:
---
### JWT 简介

`JWT`是 `json web token` 缩写。它将用户信息加密到token里，服务器不保存任何用户信息。服务器通过使用保存的密钥验证 token的正确性，只要正确即通过验证。

优点是在分布式系统中，很好地解决了单点登录问题，很容易解决了session共享的问题。 

缺点是无法作废已颁布的令牌/不易应对数据过期。

- JWT的结构

JWT包含了使用 . 分隔的三部分：

Header 头部

Payload 负载

Signature 签名

其结构看起来是这样的

```
xasdfasdfasdfxxxx.yydfdfdfdyyy.zzasdfasdfsadfasdfzzz
```

Header
在header中通常包含了两部分：token类型和采用的加密算法。

```
{
  "alg": "HS256",
  "typ": "JWT"
}  
```
接下来对这部分内容使用 Base64Url 编码组成了JWT结构的第一部分。

- Payload
Token的第二部分是负载，它包含了claim， Claim是一些实体（通常指的用户）的状态和额外的元数据，有三种类型的claim： reserved , public 和 private .

Reserved claims: 这些claim是JWT预先定义的，在JWT中并不会强制使用它们，而是推荐使用，常用的有 iss（签发者） , exp（过期时间戳） , sub（面向的用户） , aud（接收方） , iat（签发时间） 。

Public claims：根据需要定义自己的字段，注意应该避免冲突

Private claims：这些是自定义的字段，可以用来在双方之间交换信息

负载使用的例子：
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```
上述的负载需要经过 Base64Url 编码后作为JWT结构的第二部分。

- Signature

创建签名需要使用编码后的header和payload以及一个秘钥，使用header中指定签名算法进行签名。例如如果希望使用HMAC SHA256算法，那么签名应该使用下列方式创建：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)  
```

签名用于验证消息的发送者以及消息是没有经过篡改的。

- ### 完整的JWT

JWT格式的输出是以 `.` 分隔的三段Base64编码，与SAML等基于XML的标准相比，JWT在HTTP和HTML环境中更容易传递。

下列的JWT展示了一个完整的JWT格式，它拼接了之前的Header， Payload以及秘钥签名：

![](http://upload-images.jianshu.io/upload_images/11462107-438ac228458facd1.png!web?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 如何使用JWT？

在身份鉴定的实现中，传统方法是在服务端存储一个session，给客户端返回一个cookie，而使用JWT之后，当用户使用它的认证信息登陆系统之后，会返回给用户一个JWT，用户只需要本地保存该token（通常使用local storage，也可以使用cookie）即可。

当用户希望访问一个受保护的路由或者资源的时候，通常应该在 `Authorization`头部使用 `Bearer` 模式添加JWT，其内容看起来是下面这样：

```
Authorization: Bearer <token>
```

因为用户的状态在服务端的内存中是不存储的，所以这是一种 无状态 的认证机制。服务端的保护路由将会检查请求头 `Authorization` 中的JWT信息，如果合法，则允许用户的行为。由于JWT是自包含的，因此减少了需要查询数据库的需要。

JWT的这些特性使得我们可以完全依赖其无状态的特性提供数据API服务，甚至是创建一个下载流服务。因为JWT并不使用Cookie的，所以你可以使用任何域名提供你的API服务而不需要担心跨域资源共享问题（CORS）。

下面的序列图展示了该过程：

![](https://upload-images.jianshu.io/upload_images/11462107-8069112b7c0bdc6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

微服务中流程：
![](https://upload-images.jianshu.io/upload_images/11462107-ae6723be00e3dc0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
用户在提交登录信息后，服务器校验数据后将通过密钥的方式来生成一个字符串token返回给客户端，客户端在之后的请求会把token放在header里，在请求到达服务器后，服务器会检验和解密token，如果token被篡改或者失效将会拒绝请求，如果有效则服务器可以获得用户的相关信息并执行请求内容，最后将结果返回。
在微服务架构下,通常有单独一个服务Auth去管理相关认证，为了安全不会直接让用户访问某个服务，会开放一个入口服务作为网关gateway，只允许外网网关，所有请求首先访问gateway，有gateway将请求路由到各个服务。

客户端请求网关后，网关会根据路径过滤请求，是登录获取token操作的路径则直接放行，请求直接到达auth服务进行登录操作，之后进行JWT私钥加密生成token返回给客户端；是其他请求将会进行token私钥解密校验，如果token被篡改或者失效则直接拒绝访问并返回错误信息，如果验证成功经过路由到达请求服务，请求服务响应并返回数据。

- 如何实现登录、刷新、注销等？
登录比较简单，在验证身份信息后可以使用工具包例如jjwt根据用户信息生成token并设置有效时长，最后将token返回给客户端存储即可，客户端只需要每次访问时将token加在请求头里即可,然后在zuul增加一个filter,此filter来过滤请求，如果是登录获取token则放行，其他的话用公钥解密验证token是否有效。
如果要实现刷新，则需要在生成token时生成一个refreshKey，在登录时和token一并返回给客户端，然后由客户端保存定时使用refreshKey和token来刷新获取新的有效时长的token,这个refreshKey可自定义生成，为了安全起见，服务器可能需要缓存refreshKey，可使用redis来进行存储，每次刷新token都将生成新的refreshKey和token，服务器需要将老refreshKey替换，客户端保存新的token和refreshKey来进行之后的访问和刷新。
如果要实现注销，并使得旧的token即便在有效期内也不能通过验证，则需要修改登录、刷新、和优化zuul的filter。首先在登录时生成token和refreshKey后，需要将token也进行缓存，如果通过redis进行缓存可以直接放一个Set下，此Set存储所有未过期的token。其次，在刷新时在这个Set中删除旧的token并放入新的。最后对zuulFilter进行优化，在解密时先从redis里存放token的Set查找此token是否存在（redis的Set有提供方法），如果没有则直接拒绝，如果有再进行下一步解密验证有效时长，验证有效时长是为了防止刷新机制失效、没有刷新机制、网络异常强行退出等事件出现，在这种情况下旧的token没有被删除，导致了旧的token一直可以访问（如果只验证是否token是否在缓存中）。在注销时只需要删除redis中Set的token记录就好，最后写个定时器去定时删除redis中Set里面过时的token,原因也是刷新机制失效、没有刷新机制、网络异常强行退出等事件出现导致旧的token没有被删除。



### 为什么要使用JWT？

相比XML格式，JSON更加简洁，编码之后更小。
因为JSON可以直接映射为对象，在大多数编程语言中都提供了JSON解析器，而XML则没有这么自然的文档-对象映射关系。


### spring security 简介

Spring Security是一个能够为基于Spring的企业应用系统提供声明式的安全访问控制解决方案的安全框架。它提供了一组可以在Spring应用上下文中配置的Bean，充分利用了Spring IoC，DI（控制反转Inversion of Control ,DI:Dependency Injection 依赖注入）和AOP（面向切面编程）功能，为应用系统提供声明式的安全访问控制功能，减少了为企业系统安全控制编写大量重复代码的工作。

- HttpSecurity 常用方法及说明
![](https://upload-images.jianshu.io/upload_images/11462107-6791e6518b14ad58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/11462107-24307c53c0a884ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 实战

- 引入依赖
```xml
        <!--安全框架-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!--JSON封装-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.36</version>
        </dependency>

        <!--JWT-->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.0</version>
        </dependency>
```

- security 配置类 ,写一个继承于WebSecurityConfigurerAdapter的配置类,在重写带参httpsecurity,注入自定义的各种返回json的Handler

```java
@Configuration
public class MySecurityConfig extends WebSecurityConfigurerAdapter {
 @Autowired
    AjaxAuthenticationEntryPoint authenticationEntryPoint;  //  未登陆时返回 JSON 格式的数据给前端（否则为 html）

    @Autowired
    AjaxAuthenticationSuccessHandler authenticationSuccessHandler;  // 登录成功返回的 JSON 格式数据给前端（否则为 html）

    @Autowired
    AjaxAuthenticationFailureHandler authenticationFailureHandler;  //  登录失败返回的 JSON 格式数据给前端（否则为 html）

    @Autowired
    AjaxLogoutSuccessHandler logoutSuccessHandler;  // 注销成功返回的 JSON 格式数据给前端（否则为 登录时的 html）

    @Autowired
    AjaxAccessDeniedHandler accessDeniedHandler;    // 无权访问返回的 JSON 格式数据给前端（否则为 403 html 页面）

    @Autowired
    SelfUserDetailsService userDetailsService; // 自定义user

    @Autowired
    JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter; // JWT 拦截器

  @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 加入自定义的安全认证
        auth.userDetailsService(userDetailsService).passwordEncoder(new BCryptPasswordEncoder());
    }

   @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 去掉 CSRF（跨域）
        http.csrf().disable()
 .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 使用 JWT，关闭session
                .and()

                //  未登陆时返回 JSON 
                .httpBasic().authenticationEntryPoint(authenticationEntryPoint) 
                .and()

                // 所有请求必须认证
                .authorizeRequests()
                .anyRequest()
                // 认证的逻辑
                .access("@rbacauthorityservice.hasPermission(request,authentication)") // RBAC 动态 url 认证
                .and()
                //开启登录
                .formLogin()
                .loginPage("/") 
                .successHandler(authenticationSuccessHandler) // 登录成功
                .failureHandler(authenticationFailureHandler) // 登录失败
                .permitAll()
                .and()
                // 登出
                .logout()
                .logoutSuccessHandler(logoutSuccessHandler)
                .permitAll();

        // 记住我
        http.rememberMe().rememberMeParameter("remember-me")
                .userDetailsService(userDetailsService).tokenValiditySeconds(300);

        http.exceptionHandling().accessDeniedHandler(accessDeniedHandler); // 无权访问 JSON 格式的数据
        //用重写的Filter替换掉原有的UsernamePasswordAuthenticationFilter实现使用json 数据也可以登陆
        http.addFilterAt(customAuthenticationFilter(),
                UsernamePasswordAuthenticationFilter.class);
// 设置执行其他工作前的 filter （最重要的验证 JWT）
        http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class); // JWT Filter
    }

    //注册自定义的UsernamePasswordAuthenticationFilter
    @Bean
    CustomAuthenticationFilter customAuthenticationFilter() throws Exception {
        CustomAuthenticationFilter filter = new CustomAuthenticationFilter();
        filter.setAuthenticationSuccessHandler(authenticationSuccessHandler);
        filter.setAuthenticationFailureHandler(authenticationFailureHandler);
        filter.setFilterProcessesUrl("/login/self"); // 设置登陆接口名

        //这句很关键，重用WebSecurityConfigurerAdapter配置的AuthenticationManager，不然要自己组装AuthenticationManager
        filter.setAuthenticationManager(authenticationManagerBean());
        return filter;
    }
}
```
- 编写高可用对象
```java
/**
 * @Auther: Tangzhiqiang
 * @Date: 2019/1/12 16:20
 * @Description: 高可用对象那个
 */
@Data
public class AjaxResponseBody implements Serializable {
    private String status;
    private String msg;
    private Object result;
    private String jwtToken;
}
```


- 编写各返回 handler类
```java
// 没有权限处理类
@Component
public class AjaxAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException, ServletException {
        AjaxResponseBody responseBody = new AjaxResponseBody();

        responseBody.setStatus("300");
        responseBody.setMsg("需要权限!");

        httpServletResponse.getWriter().write(JSON.toJSONString(responseBody));
    }
}


// 未登陆时
@Component
public class AjaxAuthenticationEntryPoint implements AuthenticationEntryPoint {

    public void commence(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
        AjaxResponseBody responseBody = new AjaxResponseBody();

        responseBody.setStatus("000");
        responseBody.setMsg("未登录!");

        httpServletResponse.getWriter().write(JSON.toJSONString(responseBody));
    }
}

// 登录失败
@Component
public class AjaxAuthenticationFailureHandler implements AuthenticationFailureHandler {

    public void onAuthenticationFailure(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
        AjaxResponseBody responseBody = new AjaxResponseBody();

        responseBody.setStatus("400");
        responseBody.setMsg("登陆失败");

        httpServletResponse.getWriter().write(JSON.toJSONString(responseBody));
    }
}

// 登陆成功
@Component
public class AjaxAuthenticationSuccessHandler implements AuthenticationSuccessHandler {


    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        AjaxResponseBody responseBody = new AjaxResponseBody();

        responseBody.setStatus("00");
        responseBody.setMsg("登陆成功!");

        SelfUserDetails selfUserDetails = (SelfUserDetails) authentication.getPrincipal();

// 创建 token ，并返回 ，设置过期时间为 300 秒
        String jwtToken = JwtTokenUtil.generateToken(selfUserDetails.getUsername(), 300);
        responseBody.setJwtToken(jwtToken);

        response.getWriter().write(JSON.toJSONString(responseBody));
    }
}

// 登出成功
@Component
public class AjaxLogoutSuccessHandler implements LogoutSuccessHandler {

    public void onLogoutSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
        AjaxResponseBody responseBody = new AjaxResponseBody();

        responseBody.setStatus("100");
        responseBody.setMsg("登陆成功");

        httpServletResponse.getWriter().write(JSON.toJSONString(responseBody));
    }
}
```
- 最用要的 JWT 认证 filter 

  - 加密方式（对称加密，非对称加密）
    非对称加密：生成非对称加密密钥（使用jdk自带的keytool工具,注意配置好JAVA_HOME，cmd 输入以下命令就会生产密钥到cmd 所在目录）
```
keytool -genkey -alias jwt -keyalg  RSA -keysize 1024 -validity 365 -keystore jwt.jks
```

使用keytool生成密钥，别名为 `jwt`，算法为RSA，有效期为 `365 `天，文件名为jwt.jks,把文件保存在当前打开cmd的路径下,它提示输入密码,输入自定义密码，当前我设置为 `mengma` ，接下的输入可以全部忽略，回车即可，最后输入 `y` 确定，把生成的文件复制到resources目录下,写一个JwtTokenUtil 生成和解析方法。

- JwtTokenUtil 
```java
public class JwtTokenUtil {

    private static InputStream inputStream = Thread.currentThread().getContextClassLoader().getResourceAsStream("jwt.jks"); // 寻找证书文件
    private static PrivateKey privateKey = null;
    private static PublicKey publicKey = null;

    static { // 将证书文件里边的私钥公钥拿出来
        try {
            KeyStore keyStore = KeyStore.getInstance("JKS"); // java key store 固定常量
            keyStore.load(inputStream, "mengma".toCharArray());
            privateKey = (PrivateKey) keyStore.getKey("jwt", "mengma".toCharArray()); // jwt 为 命令生成整数文件时的别名
            publicKey = keyStore.getCertificate("jwt").getPublicKey();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     *
     * 使用私钥加密 token
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/13 20:43
     */
    public static String generateToken(String subject, int expirationSeconds) {
        return Jwts.builder()
                .setClaims(null)
                .setSubject(subject)
                .setExpiration(new Date(System.currentTimeMillis() + expirationSeconds * 1000))
                .signWith(SignatureAlgorithm.RS256, privateKey)
                .compact();
    }

    /**
     *
     * 不使用公钥私钥 加密token
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/13 20:41
     */
    public static String generateToken(String subject, int expirationSeconds, String salt) {
        return Jwts.builder()
                .setClaims(null)
                .setSubject(subject)
                .setExpiration(new Date(System.currentTimeMillis() + expirationSeconds * 1000))
                .signWith(SignatureAlgorithm.HS512, salt) // 不使用公钥私钥
                .compact();
    }

    /**
     *
     * 通过 公钥解密token
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/13 20:40
     */
    public static String parseToken(String token) {
        String subject = null;
        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(publicKey)
                    .parseClaimsJws(token).getBody();
            subject = claims.getSubject();
        } catch (Exception e) {
        }
        return subject;
    }

    /**
     *
     * 不嘛通过 公钥解密token
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/13 20:40
     */
    public static String parseToken(String token,String salt) {
        String subject = null;
        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(salt) // 不使用公钥私钥
                    .parseClaimsJws(token).getBody();
            subject = claims.getSubject();
        } catch (Exception e) {
        }
        return subject;
    }

}
```
- JwtAuthenticationTokenFilter
```java

/**
 * @Auther: Tangzhiqiang
 * @Date: 2019/1/13 21:18
 * @Description: OncePerRequestFilter 确保在一次请求只通过一次filter，而不需要重复执行。
 */
// TODO 还要实现 token 缓存
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    SelfUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            final String authToken = authHeader.substring("Bearer ".length());

            String username = JwtTokenUtil.parseToken(authToken);

            if (username != null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (userDetails != null) {
                    UsernamePasswordAuthenticationToken authentication =
                            new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        chain.doFilter(request, response);
    }
}
```

- 自定义 使用 json 格式登陆时filter （CustomAuthenticationFilter）
```java
/**
 * 自定义 json 登录
 */
public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {

        //attempt Authentication when Content-Type is json
        if (request.getContentType().equals(MediaType.APPLICATION_JSON_UTF8_VALUE)
                || request.getContentType().equals(MediaType.APPLICATION_JSON_VALUE)) {



            UsernamePasswordAuthenticationToken authRequest = null;
            try {
                String jsonString = GetRequestJsonUtils.getPostRequestJsonString(request);
                SelfUserDetails selfUserDetails = JsonUtils.jsonToPojo(jsonString,SelfUserDetails.class);
                authRequest = new UsernamePasswordAuthenticationToken(
                        selfUserDetails.getUsername(), selfUserDetails.getPassword());
            } catch (Exception e) {
                e.printStackTrace();
                authRequest = new UsernamePasswordAuthenticationToken(
                        "", "");
            } finally {
                setDetails(request, authRequest);
                return this.getAuthenticationManager().authenticate(authRequest);
            }
        }
        //transmit it to UsernamePasswordAuthenticationFilter
        else {
            return super.attemptAuthentication(request, response);
        }
    }
}
```
- 获取 post 请求 request 中的 json数据工具类 （GetRequestJsonUtils）
```java
public class GetRequestJsonUtils {

    // 返回 json  字符串
    public static String getPostRequestJsonString(HttpServletRequest request) {
        BufferedReader br;
        StringBuilder sb = null;
        String jsonString = null;
        try {
            br = new BufferedReader(new InputStreamReader(
                    request.getInputStream()));
            String line = null;
            sb = new StringBuilder();
            while ((line = br.readLine()) != null) {
                sb.append(line);
            }
            jsonString = URLDecoder.decode(sb.toString(), "UTF-8");
            jsonString = jsonString.substring(jsonString.indexOf("{"));
        } catch (IOException e) {
            e.printStackTrace();
        }
        return jsonString;
    }
}
```

- JsonUtils 工具类
```java
public class JsonUtils {

    // 定义jackson对象
    private static final ObjectMapper MAPPER = new ObjectMapper();

    /**
     * 将对象转换成json字符串。
     * <p>Title: pojoToJson</p>
     * <p>Description: </p>
     * @param data
     * @return
     */
    public static String objectToJson(Object data) {
        try {
            String string = MAPPER.writeValueAsString(data);
            return string;
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return null;
    }
    
    /**
     * 将json结果集转化为对象
     * 
     * @param jsonData json数据
     * @param beanType 对象中的object类型
     * @return
     */
    public static <T> T jsonToPojo(String jsonData, Class<T> beanType) {
        try {
            T t = MAPPER.readValue(jsonData, beanType);
            return t;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
    
    /**
     * 将json数据转换成pojo对象list
     * <p>Title: jsonToList</p>
     * <p>Description: </p>
     * @param jsonData
     * @param beanType
     * @return
     */
    public static <T>List<T> jsonToList(String jsonData, Class<T> beanType) {
        JavaType javaType = MAPPER.getTypeFactory().constructParametricType(List.class, beanType);
        try {
            List<T> list = MAPPER.readValue(jsonData, javaType);
            return list;
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        return null;
    }
    
}
```
- 用户访问权限具体判断逻辑（RbacAuthorityService）
```java
@Component("rbacauthorityservice")
public class RbacAuthorityService {

    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {

        Object userInfo = authentication.getPrincipal();

        boolean hasPermission  = false;


        if (userInfo instanceof UserDetails) {

            String username = ((UserDetails) userInfo).getUsername();

            Collection<? extends GrantedAuthority> authorities = ((UserDetails) userInfo).getAuthorities();
            Iterator<? extends GrantedAuthority> iterator = authorities.iterator();
            for (GrantedAuthority authority : authorities) {
                if (authority.getAuthority().equals("ROLE_ADMIN")) {

                    //admin 可以访问的资源
                    Set<String> urls = new HashSet();
                    urls.add("/sys/**");
                    urls.add("/test/**");
                    AntPathMatcher antPathMatcher = new AntPathMatcher();
                    for (String url : urls) {
                        if (antPathMatcher.match(url, request.getRequestURI())) {
                            hasPermission = true;
                            break;
                        }
                    }
                }
            }
            //user可以访问的资源
            Set<String> urls = new HashSet();
            urls.add("/test/**");
            AntPathMatcher antPathMatcher = new AntPathMatcher();
            for (String url : urls) {
                if (antPathMatcher.match(url, request.getRequestURI())) {
                    hasPermission = true;
                    break;
                }
            }
            return hasPermission;
        } else {
            return false;
        }


    }
}
```

### 未完持续。。。。。

