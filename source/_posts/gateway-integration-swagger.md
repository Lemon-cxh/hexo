---
title: Spring Cloud Gateway集成Swagger
date: 2019-12-13 17:36:55
categories: 
- SpringCloud
tags:
- Swagger
- Spring Cloud
description: Spring Cloud Gateway集成Swagger，以及Nginx配置
---
1. #### 微服务中配置多组文档接口
    ```java
    @EnableSwagger2
    @Configuration
    public class RESTFulWebAPI {

        @Bean
        public Docket createControllerApi() {
            return new Docket(DocumentationType.SWAGGER_2)
                    .groupName("接口分组名1")
                    .apiInfo(apiInfo())
                    .select()
                    .apis(RequestHandlerSelectors.basePackage("com.**.**.controller"))
                    .paths(PathSelectors.any())
                    .build()
                    .securitySchemes(securitySchemes())
                    .securityContexts(securityContexts());
        }

        @Bean
        public Docket createRpcApi() {
            return new Docket(DocumentationType.SWAGGER_2)
                    .groupName("接口分组名2")
                    .apiInfo(apiInfo())
                    .select()
                    .apis(RequestHandlerSelectors.basePackage("com.**.**.rpc"))
                    .paths(PathSelectors.any())
                    .build()
                    // 设置全局token,解决接口需要token验证的问题
                    .securitySchemes(securitySchemes())
                    .securityContexts(securityContexts());
        }

        private List<ApiKey> securitySchemes() {
            List<ApiKey> apiKeyList = new ArrayList();
            apiKeyList.add(new ApiKey("Authorization", "Authorization", "header"));
            return apiKeyList;
        }

        private List<SecurityContext> securityContexts() {
            List<SecurityContext> securityContexts = new ArrayList<>();
            securityContexts.add(
                    SecurityContext.builder()
                            .securityReferences(defaultAuth())
                            .forPaths(PathSelectors.regex("^(?!auth).*$"))
                            .build());
            return securityContexts;
        }

        private List<SecurityReference> defaultAuth() {
            AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
            AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
            authorizationScopes[0] = authorizationScope;
            List<SecurityReference> securityReferences = new ArrayList<>();
            securityReferences.add(new SecurityReference("Authorization", authorizationScopes));
            return securityReferences;
        }

        private ApiInfo apiInfo() {
            return new ApiInfoBuilder()
                    .title("文档标题")
                    .version("0.0.0.1")
                    .description("API描述")
                    .build();
        }
    }
    ```
2. #### Gateway配置网关
	1. ##### pom文件添加依赖
		```xml
		<dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
		```
	2. ##### 注入路由到SwaggerResource
		```java
        @Component
        @Primary
        @AllArgsConstructor
        public class SwaggerProvider implements SwaggerResourcesProvider {

            private final RouteLocator routeLocator;
            private final GatewayProperties gatewayProperties;
            private final RestTemplate restTemplate;

            @Override
            public List<SwaggerResource> get() {
                List<SwaggerResource> resources = new ArrayList<>();
                List<String> routes = new ArrayList<>();
                //取出gateway的route
                routeLocator.getRoutes().subscribe(route -> routes.add(route.getId()));
                //结合配置的route-路径(Path)，和route过滤，只获取有效的route节点
                gatewayProperties.getRoutes().stream().filter(routeDefinition -> routes.contains(routeDefinition.getId()))
                        .forEach(routeDefinition -> routeDefinition.getPredicates().stream()
                                .filter(predicateDefinition -> ("Path").equalsIgnoreCase(predicateDefinition.getName()))
                                .forEach(predicateDefinition -> {
                                    SwaggerResource[] swaggerResources;
                                    try {
                                        swaggerResources = restTemplate
                                                .getForObject("http://" + routeDefinition.getId() + "/swagger-resources", SwaggerResource[].class);
                                    } catch (HttpClientErrorException | IllegalStateException e) {
                                        return;
                                    }
                                    if (Objects.isNull(swaggerResources)) {
                                        return;
                                    }
                                    for (SwaggerResource resource : swaggerResources) {
                                        resource.setUrl(predicateDefinition.getArgs().get(NameUtils.GENERATED_NAME_PREFIX + "0")
                                                .replace("/**", resource.getUrl()));
                                        resources.add(resource);
                                    }
                                }));
                return resources;
            }
        }
		```
	3. ##### 提供Swagger对外接口
		```java
		@RestController
		@RequestMapping("/swagger-resources")
		public class SwaggerController {
		
		    @Autowired(required = false)
		    private SecurityConfiguration securityConfiguration;
		    @Autowired(required = false)
		    private UiConfiguration uiConfiguration;
		    private final SwaggerResourcesProvider swaggerResources;
		
		
		    @GetMapping("/configuration/security")
		    public Mono<ResponseEntity<SecurityConfiguration>> securityConfiguration() {
		        return Mono.just(new ResponseEntity<>(
		                Optional.ofNullable(securityConfiguration).orElse(SecurityConfigurationBuilder.builder().build()), HttpStatus.OK));
		    }
		
		    @GetMapping("/configuration/ui")
		    public Mono<ResponseEntity<UiConfiguration>> uiConfiguration() {
		        return Mono.just(new ResponseEntity<>(
		                Optional.ofNullable(uiConfiguration).orElse(UiConfigurationBuilder.builder().build()), HttpStatus.OK));
		    }
		
		    @GetMapping
		    public Mono<ResponseEntity> swaggerResources() {
		        return Mono.just((new ResponseEntity<>(swaggerResources.get(), HttpStatus.OK)));
		    }
		
		    @Autowired
		    public SwaggerController(SwaggerResourcesProvider swaggerResources) {
		        this.swaggerResources = swaggerResources;
		    }
		
		}
		```
	4. ##### Swagger的路径转换

        Swagger文档中的路径为：`主机名:端口:映射路径`少了一个服务路由前缀，是因为展示handler经过了`StripPrefixGatewayFilterFactory`这个过滤器的处理，原有的路由前缀被过滤掉了！
		- ###### 方案1：通过Swagger的host配置手动维护一个前缀
			```java
			return new Docket(DocumentationType.SWAGGER_2)
			    .apiInfo(apiInfo())
			    .host("主机名：端口：服务前缀")  //注意这里的主机名：端口是网关的地址和端口
			    .select()
			    .apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
			    .paths(PathSelectors.any())
			    .build()
			    .globalOperationParameters(parameterList);
			```
		- ###### 方案2：增加X-Forwarded-Prefix
			```java
			@Component
			public class SwaggerHeaderFilter extends AbstractGatewayFilterFactory {

			    private static final String HEADER_NAME = "X-Forwarded-Prefix";
                private static final String API_URI = "/v2/api-docs";
			
			    @Override
			    public GatewayFilter apply(Object config) {
			        return (exchange, chain) -> {
			            ServerHttpRequest request = exchange.getRequest();
			            String path = request.getURI().getPath();
			            if (!StringUtils.endsWithIgnoreCase(path, API_URI)) {
			                return chain.filter(exchange);
			            }
			            String basePath = path.substring(0, path.lastIndexOf(API_URI));
			            ServerHttpRequest newRequest = request.mutate().header(HEADER_NAME, basePath).build();
			            ServerWebExchange newExchange = exchange.mutate().request(newRequest).build();
			            return chain.filter(newExchange);
			        };
			    }
			}
			```
			配置appction.yml
			```yml
			- id: test
			  uri: lb://test
			  predicates:
			  - Path=/test/**
			  filters:
			  - SwaggerHeaderFilter
			  - StripPrefix=1
			```
3. #### Nginx配置
	1. ##### Nginx配置文件
		```conf
			# 配置使用用户名和密码登录
			location /swagger-ui.html {
		        auth_basic "test";
		        # 相对路径：htpasswd在机器上的位置：/usr/local/nginx/conf/htpasswd
		        auth_basic_user_file htpasswd;
		        # 绝对路径：htpasswd在机器上的位置：/tmp/htpasswd
		        # auth_basic_user_file /tmp/htpasswd;
		    	proxy_pass http://***/swagger-ui.html;
		    }
		
		    location /swagger-resources {
		    	proxy_pass http://**/swagger-resources;
		    }
		
		    location /webjars {
		    	proxy_pass http://**/webjars;
		    }
		```
	2. ##### 创建htpasswd
		```
		# 格式：用户名:密码，注意密码是使用crypt加密过的
		admin:PbSRr7orsxaso
		```

4. #### 参考
    - [Spring Cloud Gateway 聚合swagger文档](https://juejin.im/post/5b511d016fb9a04fda4e13cb#heading-0)
    - [nginx配置指令auth_basic、auth_basic_user_file及相关知识](https://www.jianshu.com/p/1c0691c9ad3c)