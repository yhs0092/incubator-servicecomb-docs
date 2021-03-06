* **Q: 用IntelliJ的免费版开发，有什么问题？**   

    * A: 没有问题，使用IntelliJ 开发，可参考 [Setup Developer Environment](/cn/developers/setup-develop-environment/) 进行相应的环境配置。    

* **Q: 契约生成会报错Caused by: java.lang.Error: OperationId must be unique，不支持函数重载？**

    * A: 支持函数重载，但是需要注意每个接口必须有唯一的operation id。可以加上`@ApiOperation`标签给重载的接口指定唯一的 operation id。

* **Q: 使用*spring-boot-starter-provider*这个依赖时，在*application.yml*文件中声明的`spring.main.web-application`属性并没有生效？**

    * A:  最近版本只支持 spring boot 2， 可以通过 ApplicationBuilder 创建运行环境。 参考[在Spring Boot中使用java chassis](../using-java-chassis-in-spring-boot/using-java-chassis-in-spring-boot.md)。
 
* **Q: 如何自定义某个Java方法对应的REST接口里的HTTP Status Code？**

    * A:  对于正常的返回值，可以通过SwaggerAnnotation实现，例如：

            @ApiResponse(code = 300, response = String.class, message = "")
            public int test(int x) {
              return 100;
            }

        对于异常的返回值，可以通过抛出自定义的InvocationException实现，例如：

            public String testException(int code) {
                String strCode = String.valueOf(code);
                switch (code) {
                    case 200:
                        return strCode;
                    case 456:
                        throw new InvocationException(code, strCode, strCode + " error");
                    case 556:
                        throw new InvocationException(code, strCode, Arrays.asList(strCode + " error"));
                    case 557:
                        throw new InvocationException(code, strCode, Arrays.asList(Arrays.asList(strCode + " error")));
                    default:
                        break;
                }
                return "not expected";
            }

* **Q: 如何定制自己微服务的日志配置?**

    * A: ServiceComb不绑定日志器，只是使用了slf4j，用户可以自由选择log4j/log4j2/logback等等。ServiceComb提供了一个log4j的扩展，在标准log4j的基础上，支持log4j的properties文件的增量配置。
        * 默认以规则："classpath\*:config/log4j.properties"加载配置文件
        * 实际会搜索出classpath中所有的```config/log4j.properties和config/log4j.*.properties```, 从搜索出的文件中切出```\*```的部分，进行alpha排序，然后按顺序加载，最后合成的文件作为log4j的配置文件。
        * 如果要使用ServiceComb的log4j扩展，则需要调用Log4jUtils.init，否则完全按标准的日志器的规则使用。

* **Q: 当服务配置了多个transport的时候，在运行时是怎么选择使用哪个transport的？**

    * A: ServiceComb的consumer、transport、handler、producer之间是解耦的，各功能之间通过契约定义联合在一起工作的，即：
          consumer使用透明rpc，还是springmvc开发与使用highway，还是RESTful在网络上传输没有关系与producer是使用透明rpc，还是jaxrs，
          或者是springmvc开发，也没有关系handler也不感知，业务开发方式以及传输方式。consumer访问producer，在运行时的transport选择上，
          总规则为：cnsumer的transport与producer的endpoint取交集，如果交集后，还有多个transport可选择，则轮流使用，分解开来，存在以下场景：

        * 当一个微服务producer同时开放了highway以及RESTful的endpoint
            * consumer进程中只部署了highway transport jar，则只会访问producer的highway endpoint
            * consumer进程中只部署了RESTful transport jar，则只会访问producer的RESTful endpoint
            * consumer进程中，同时部署了highway和RESTful transport jar，则会轮流访问producer的highway、RESTful endpoint

         * 如果consumer想固定使用某个transport访问producer，可以在consumer进程的microservice.yaml中配置，指定transport的名称:

                servicecomb:
                    references:
                        <service_name>:
                            transport: highway

         * 当一个微服务producer只开放了highway的endpoint
             * consumer进程只部署了highway transport jar，则正常使用highway访问
             * consumer进程只部署了RESTful transport jar，则无法访问
             * consumer进程同时部署了highway和RESTful transport jar，则正常使用highway访问

         * 当一个微服务producer只开放了RESTful的endpoint
             * consumer进程只部署了highway transport jar，则无法访问
             * consumer进程只部署了RESTful transport jar，则正常使用RESTful访问
             * consumer进程同时部署了highway和RESTful transport jar，则正常使用RESTful访问

* **Q: ServiceComb微服务框架服务调用是否使用长连接?**

    * A: http使用的是长连接（有超时时间），highway方式使用的是长连接（一直保持）。

* **Q: 服务断连服务中心注册信息是否自动删除**

    * A: 服务中心心跳检测到服务实例不可用，只会移除服务实例信息，服务的静态数据不会移除。

* **Q: 如果使用tomcat方式集成ServiceComb微服务框架，如何实现服务注册**

    * A: 如果使用ServiceComb sdk servlet方式（使用transport-rest-servlet依赖）制作为war包部署到tomcat，需要保证，
        服务描述文件（microservice.yaml）中rest端口配置和外置容器一致才能实现该服务的正确注册。否则无法感知tomcat开放端口。
 
* **Q: 如果使用tomcat方式集成ServiceComb微服务框架，服务注册的时候如何将war包部署的上下文注册到服务中心**

    * A: 发布服务接口的时候需要将war包部署的上下文（context）放在baseurl最前面，这样才能保证注册到服务中心的路径是完整的路径（包含了上下文）。示例：

            @path(/{context}/xxx)
            class ServiceA
 
* **Q:  ServiceComb微服务框架如何实现数据多个微服务间透传**

    * A: 透传数据塞入：

            CseHttpEntity<xxxx.class> httpEntity = new CseHttpEntity<>(xxx);
            //透传内容
            httpEntity.addContext("contextKey","contextValue");
            ResponseEntity<String> responseEntity = RestTemplateBuilder.create()
                .exchange("cse://springmvc/springmvchello/sayhello",HttpMethod.POST,httpEntity,String.class);

        透传数据获取：

            @Override
            @RequestMapping(path="/sayhello",method = RequestMethod.POST)
            public String sayHello(@RequestBody Person person,InvocationContext context){
                 //透传数据获取
                 context.getContext();
                 return "Hello person " + person.getName();
            }

* **Q:  ServiceComb body Model部分暴露**

    * A: 一个接口对应的body对象中，可能有一些属性是内部的，不想开放出去，生成schema的时候不要带出去，使用：

            @ApiModelProperty(hidden = true)

* **Q:  服务超时设置**

    * A: 在微服务描述文件（microservice.yaml）中添加如下配置：

            servicecomb:
                request:
                    timeout: 30000
   
* **Q:  URL 地址就可以唯一定位，为什么要加上一个schema？**

    * A: 
        1. schema 是用来匹配服务契约的，用来保证服务端和消费端契约兼容，每个契约需要一个唯一ID，在服务中心存储。
        2. schema映射到java的interface概念，在consumer使用透明rpc模式开发时，可以找到是微服务里的哪个operation。schema之间的方法名是没有唯一性要求的。
        3. operation qualified name是治理的key，而URL 因为path参数的存在，没办法直接查找，而qualified name是不会变的。治理是不区分传输的，如果治理按URL 走，那么highway调进来时，还得根据参数反向构造出url，再来正则表达式匹配，太折腾了。
        4. http只是一种传输通道，还有别的传输通道不需要映射到URL的。

* **Q: rest客户端调用的时候，实际上只带上了服务名和URL，并不需要指定schema id的， 而实际上根据这个URL也能找到具体契约的，所以指定schema id作用何在？**

    * A: 由于透明rpc是接口式调用，并没有URL，内部实际都归一化到operation来描述的，这样就可以结合schema id唯一定位到具体的请求处理中。

* **Q:  Transport是个什么概念？用来干什么的？**

    * A: transport负责编解码，以及传输。通信模型有rest和highway两种，highway对应的是私有协议，使用protobuf编码，rest用的是json。highway和rest都是基于vertx做的，vertx是基于netty的。

* **Q:  ServiceComb和服务中心是怎么交互的?**

    * A: 走rest，主要负责注册，取数据和心跳等；watch事件走websocket，watch事件是观察服务中心实例信息有没有变更。

* **Q:  有类似dubbo那种治理中心吗？**

    * A: bizkeeper是一个handler，是治理的其中一个内容。治理可以通过handler扩展。

* **Q: service path怎么理解？**

    * A: 每个微服务有一个servicePathManager，每一个schema将自己的path注册进去。

* **Q:  浏览器能直接访问微服务Endpoint吗？**

    * A: 可以，restful发布的微服务Endpoint，可以直接在浏览器中使用HTTP加service path访问提供get方法的服务，如果是访问其他Http方法提供的服务建议安装使用[Postman](https://www.sap.com/developer/tutorials/api-tools-postman-install.html)。

* **Q:  契约生成时，需要强制带上版本号和语言吗？**

    * A: 契约是属于微服务的，微服务本来就有版本，但语言是不应该带上版本号的。应该契约要求与语言无关。契约“没有版本”，契约的版本体现在微服务上，实例能找到所属的微服务的版本，就能找到一个确定的契约。

* **Q: 如果同时引入了`transport-rest-servlet`和`transport-rest-vertx`的依赖，那么它怎么决定采用哪一个？**

    * A: 如果端口没被占用，就用vertx；如果被占用了，就用servlet。


* **Q:  qps流控设计时是出于什么场景考虑的？**

    * A: 限流有两个主要作用，第一通过给不同的消费者限流保证对一些重点服务的服务效果，第二防止雪崩效应。可根据服务的重要性来决定水管的粗细，ServiceComb是支持消费端限流和服务端限流两种限流方式的，消费端限流可以做到比较精细的控制。
  

* **Q: 如果服务端是链式调用，即类似a->b->c，那设置了qps 流控会不会造成水管粗细不均的事情？**

    * A: 一般采取的模式是先测量再设置。qps设置最终是结合整体业务需求来进行调控的，而不是就单个节点来进行设置。

* **Q: 如何在契约DTO中忽略中指定的属性？**

    * A: 可以使用@JsonIgnore注解标记需要忽略的属性， 例如:

            public class OutputForTest{
                @JsonIgnore
                private String outputId = null;
                private String inputId = null;
                ...
             }
