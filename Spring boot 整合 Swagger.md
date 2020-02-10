title: "Spring boot 整合 Swagger"
date: 2019-07-28 10:48:16
categories: java
tags: [spring boot, swagger]

----



#### 本文github位置：<https://github.com/WillVi/springboot-swagger2-demo>

## 环境准备

1. JDK版本：1.8
2. Spring boot版本：1.5.16
3. Swagger及其Swagger-ui版本：2.6.1（关于swagger-ui版本 每个版本的汉化方式可能有不同）
4. 默认url：<http://localhost:8080/swagger-ui.html>

## Maven依赖

```xml
<!-- swagge2 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.6.1</version>
</dependency>
<!-- swaggerui用于展示swagger页面 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.6.1</version>
</dependency>
```

<!-- more -->

### 注意事项：

swagger-ui依赖 有可能会与`com.google.guava`冲突 提供一个解决办法:

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.2.2</version>
    <!-- 将冲突包移除 -->
    <exclusions>
        <exclusion>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 配置文件

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {
    @Bean
    public Docket api() {
        // 统一设置返回描述
        List<ResponseMessage> responseMessages = new ArrayList<>();
        responseMessages.add(new ResponseMessageBuilder().code(400).message("请求参数有误").build());
        responseMessages.add(new ResponseMessageBuilder().code(401).message("未授权").build());
        responseMessages.add(new ResponseMessageBuilder().code(403).message("禁止访问").build());
        responseMessages.add(new ResponseMessageBuilder().code(404).message("请求路径不存在").build());
        responseMessages.add(new ResponseMessageBuilder().code(500).message("服务器内部错误").responseModel(new ModelRef("string")).build());

        return new Docket(DocumentationType.SWAGGER_2)
                // 禁用默认返回描述
                .useDefaultResponseMessages(false)
                // 启用新的返回描述
                .globalResponseMessage(RequestMethod.GET, responseMessages)
                .globalResponseMessage(RequestMethod.POST, responseMessages)
                .globalResponseMessage(RequestMethod.PATCH, responseMessages)
                .globalResponseMessage(RequestMethod.PUT, responseMessages)
                .globalResponseMessage(RequestMethod.DELETE, responseMessages)
                // 设置基本信息
                .apiInfo(apiInfo())
                .select()
                // 设置哪些api被扫描
                .apis(RequestHandlerSelectors.basePackage("cn.willvi.springbootswaggerdemo"))
                // 设置路径
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 设置基本信息
     * @return
     */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                // 标题
                .title("我的接口文档")
            	// 描述
                .description("这是我的第一个接口文档")
            	// 版本号
                .version("1.0")
                // 项目的链接
                .termsOfServiceUrl("")
                // 作者
                .license("willvi")
                .licenseUrl("")
                .build();
    }
}
```

## Swagger注解详解

### @Api 设置Controller整体信息

注解位置：Controller类上

```java
/**
 * value                 url路径值（汉化后不起作用） http://xxx/swagger-ui.html#/demo-controller中的 demo-controller即为value
 * tags                  设置这个值、value的值会被覆盖（汉化后有bug最好不写）
 * description           对api资源的描述
 * basePath              基本路径可以不配置
 * position              如果配置多个Api 想改变显示的顺序位置
 * produces              例如, "application/json, application/xml"  页面上的 Response Content Type (响应Content Type) 约束响应资源的表现形式
 * consumes              例如, "application/json, application/xml" 请求资源的表现形式
 * protocols             Possible values: http, https, ws, wss.
 * authorizations        高级特性认证时配置
 * hidden                配置为true 将在文档中隐藏
 */
@Api(value = "Swagger 注解使用",description = "123")
```

### @ApiOperation 设置每个方法（接口）信息

注解位置：方法上

```java
  /**
     * value               页面tab右侧值
     * tags                如果设置这个值、value的值会被覆盖
     * description         对api资源的描述
     * basePath            基本路径可以不配置
     * position            如果配置多个Api 想改变显示的顺序位置
     * produces            例如, "application/json, application/xml"
     * consumes            例如, "application/json, application/xml"
     * protocols           Possible values: http, https, ws, wss.
     * authorizations      高级特性认证时配置
     * hidden              配置为true 将在文档中隐藏
     * response            返回的对象
     * responseContainer   这些对象是有效的 "List", "Set" or "Map".，其他无效
     * httpMethod  "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH"
     * code                http的状态码 默认 200
     * extensions          扩展属性
     */
     @ApiOperation(
            value = "post",
            notes = "这是一个小demo",
            produces = "application/json",
            response = Person.class
    )
```

### @ApiImplicitParams 与 @ApiImplicitParam 设置传入参数信息

注解位置：方法上

注意事项： **name必须与参数名相同，不然多出一个参数框**

```java
    /**
     * @ApiImplicitParam
     * 当dataType不是对象时可以使用
     *
     * paramType：参数放在哪个地方 path,query,heard,body,from
     * name：参数代表的含义
     * value：参数名称
     * dataType： 参数类型，有String/int，无用
     * required ： 是否必要
     * defaultValue：参数的默认值
     */
    @ApiImplicitParams(
            @ApiImplicitParam(name = "name",value = "name",paramType ="path", dataType = "String")
    )
```

### @ApiResponses 设置返回的一些状态码信息

注解位置：方法上

```java
 /**
     * code                http的状态码
     * message             描述
     * response            默认响应类 Void
     * reference           参考ApiOperation中配置
     * responseHeaders     参考 ResponseHeader 属性配置说明
     * responseContainer   参考ApiOperation中配置
     */
@ApiResponses({@ApiResponse(code = 500, message = "服务器内部错误", response = String.class)})
```

### @ApiModel 

描述一个Model的信息（这种一般用在post创建的时候，使用@RequestBody这样的场景，请求参数无法使用@ApiImplicitParam注解进行描述的时候

注解位置：实体类上

#### @ApiModelProperty

描述一个model的属性。

注解位置：实体类每个属性上面

```java
@ApiModelProperty(value = "姓名",name = "name")
```

### @ApiParam

注解位置：方法参数内

```java
 /**
     * name	属性名称
     * value	属性值
     * defaultValue	默认属性值
     * allowableValues	可以不配置
     * required	是否属性必填
     * access	不过多描述
     * allowMultiple	默认为false
     * hidden	隐藏该属性
     * example	举例子
     */
 public ResponseEntity<String> placeOrder(
      @ApiParam(value = "xxxx", required = true) Person person) {
    storeData.add(order);
    return ok("");
  }
```



## Controler示例

```java
@Api(value = "Swagger 注解使用")

@RestController
@RequestMapping("/")


public class DemoController {
    @PostMapping("/postHello")
    /**
     * value               url的路径值
     * tags                如果设置这个值、value的值会被覆盖
     * description         对api资源的描述
     * basePath            基本路径可以不配置
     * position            如果配置多个Api 想改变显示的顺序位置
     * produces            例如, "application/json, application/xml"
     * consumes            例如, "application/json, application/xml"
     * protocols           Possible values: http, https, ws, wss.
     * authorizations      高级特性认证时配置
     * hidden              配置为true 将在文档中隐藏
     * response            返回的对象
     * responseContainer   这些对象是有效的 "List", "Set" or "Map".，其他无效
     * httpMethod  "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH"
     * code                http的状态码 默认 200
     * extensions          扩展属性
     */
    @ApiOperation(
            value = "post",
            notes = "这是一个小demo",
            produces = "application/json",
            response = Person.class
    )
    /**
     * code                http的状态码
     * message             描述
     * response            默认响应类 Void
     * reference           参考ApiOperation中配置
     * responseHeaders     参考 ResponseHeader 属性配置说明
     * responseContainer   参考ApiOperation中配置
     */
    @ApiResponses({@ApiResponse(code = 500, message = "服务器内部错误", response = String.class)})
    public ResponseEntity<Person> postHello(@RequestBody Person person){
        return ResponseEntity.ok(person);
    }

    @GetMapping("/hello/{name}")
    @ApiOperation(
            value = "hello world",
            notes = "这是一个小demo"
    )
    /**
     * @ApiImplicitParam
     * 当dataType不是对象时可以使用
     * paramType：参数放在哪个地方
     * name：参数代表的含义
     * value：参数名称
     * dataType： 参数类型，有String/int，无用
     * required ： 是否必要
     * defaultValue：参数的默认值
     */
    @ApiImplicitParams(
            @ApiImplicitParam(value = "name",paramType ="path", dataType = "String",defaultValue = "world")
    )
    public String hello(@PathVariable String name){
        return "hello " + name;
    }
}
```

## Model示例

```java
// 描述一个Model的信息
@ApiModel
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Person {
    @ApiModelProperty(value = "姓名",name = "name")
    String name;
    int age;
}
```

## swagger汉化

1. 在resources资源下创建   `META-INF/resources` 文件夹并创建名为`swagger-ui.html`文件

2. 复制以下内容：

   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <meta charset="UTF-8">
       <title>Swagger UI</title>
       <link rel="icon" type="image/png" href="webjars/springfox-swagger-ui/images/favicon-32x32.png" sizes="32x32"/>
       <link rel="icon" type="image/png" href="webjars/springfox-swagger-ui/images/favicon-16x16.png" sizes="16x16"/>
       <link href='webjars/springfox-swagger-ui/css/typography.css' media='screen' rel='stylesheet' type='text/css'/>
       <link href='webjars/springfox-swagger-ui/css/reset.css' media='screen' rel='stylesheet' type='text/css'/>
       <link href='webjars/springfox-swagger-ui/css/screen.css' media='screen' rel='stylesheet' type='text/css'/>
       <link href='webjars/springfox-swagger-ui/css/reset.css' media='print' rel='stylesheet' type='text/css'/>
       <link href='webjars/springfox-swagger-ui/css/print.css' media='print' rel='stylesheet' type='text/css'/>
   
       <script src='webjars/springfox-swagger-ui/lib/object-assign-pollyfill.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/jquery-1.8.0.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/jquery.slideto.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/jquery.wiggle.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/jquery.ba-bbq.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/handlebars-4.0.5.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/lodash.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/backbone-min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/swagger-ui.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/highlight.9.1.0.pack.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/highlight.9.1.0.pack_extended.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/jsoneditor.min.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/marked.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lib/swagger-oauth.js' type='text/javascript'></script>
   
       <script src='webjars/springfox-swagger-ui/springfox.js' type='text/javascript'></script>
       <!-- 汉化 -->
       <script src='webjars/springfox-swagger-ui/lang/translator.js' type='text/javascript'></script>
       <script src='webjars/springfox-swagger-ui/lang/zh-cn.js' type='text/javascript'></script>
   </head>
   
   <body class="swagger-section">
   <div id='header'>
       <div class="swagger-ui-wrap">
           <a id="logo" href="http://swagger.io"><img class="logo__img" alt="swagger" height="30" width="30" src="webjars/springfox-swagger-ui/images/logo_small.png" /><span class="logo__title">swagger</span></a>
           <form id='api_selector'>
               <div class='input'>
                   <select id="select_baseUrl" name="select_baseUrl"/>
               </div>
               <div class='input'><input placeholder="http://example.com/api" id="input_baseUrl" name="baseUrl" type="text"/></div>
               <div id='auth_container'></div>
               <div class='input'><a id="explore" class="header__btn" href="#" data-sw-translate>Explore</a></div>
           </form>
       </div>
   </div>
   
   <div id="message-bar" class="swagger-ui-wrap" data-sw-translate>&nbsp;</div>
   <div id="swagger-ui-container" class="swagger-ui-wrap"></div>
   </body>
   </html>
   ```

3. 访问即可