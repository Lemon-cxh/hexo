---
title: 'Spring Cloud Gateway灰度负载均衡'
date: 2020-10-28 13:58:59
categories: 
- Spring Cloud
tags:
- Spring Cloud
description: 创建自定义负载均衡拦截器，根据请求头参数version来轮询调用对应服务
---
1. ##### 创建自定义负载均衡拦截器

    ```java
    public class GrayscaleLoadBalancerClientFilter extends LoadBalancerClientFilter {

        private static final String VERSION = "version";
        private final DiscoveryClient discoveryClient;
        private final AtomicInteger nextServerCyclicCounter;

        public GrayscaleLoadBalancerClientFilter(LoadBalancerClient loadBalancer, DiscoveryClient discoveryClient) {
            super(loadBalancer);
            this.discoveryClient = discoveryClient;
            this.nextServerCyclicCounter = new AtomicInteger(0);
        }

        @Override
        protected ServiceInstance choose(ServerWebExchange exchange) {
            URI uri = (URI) exchange.getAttributes().get(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
            List<ServiceInstance> list = discoveryClient.getInstances(uri.getHost());
            if (CollectionUtils.isEmpty(list)) {
                return null;
            }
            List<String> listHeaders = exchange.getRequest().getHeaders().get(VERSION);
            String version = CollectionUtils.isEmpty(listHeaders)
                    ? null : (StringUtils.isBlank(listHeaders.get(0)) ? null : listHeaders.get(0));
            if (Objects.nonNull(version)) {
                list = list.stream().
                        filter(serviceInstance -> version.equals(serviceInstance.getMetadata().get(VERSION)))
                        .collect(Collectors.toList());
                if (!CollectionUtils.isEmpty(list)) {
                    return list.get(this.incrementAndGetModulo(list.size()));
                }
            }
            list = list.stream().
                    filter(serviceInstance -> !serviceInstance.getMetadata().containsKey(VERSION))
                    .collect(Collectors.toList());
            return list.isEmpty() ? null : list.get(this.incrementAndGetModulo(list.size()));
        }

        private int incrementAndGetModulo(int modulo) {
            int current;
            int next;
            do {
                current = this.nextServerCyclicCounter.get();
                next = (current + 1) % modulo;
            } while(!this.nextServerCyclicCounter.compareAndSet(current, next));

            return next;
        }
    }
    ```

2. ##### 注入自定义负载均衡拦截器

    ```java
        @Bean
        @Profile("dev")
        public LoadBalancerClientFilter loadBalancerClientFilter(LoadBalancerClient client,
                                                                DiscoveryClient discoveryClient) {
            return new GrayscaleLoadBalancerClientFilter(client, discoveryClient);
        }
    ```

3. ##### 配置服务元数据

    可以在`bootstrap.yml`中配置，或者在注册中心中设置
    ```yml
    spring:
        cloud:
            nacos:
                discovery:
                    metadata: 
                        version: 1
    ```