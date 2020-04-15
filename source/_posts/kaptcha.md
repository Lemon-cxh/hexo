---
title: SpringBoot使用kaptcha生成验证码
date: 2019-12-24 15:44:58
categories: 
- Spring Boot
tags:
- kaptcha
- Spring Boot
---
1. ##### 添加Maven依赖
    ```xml
    <dependency>
        <groupId>com.github.penggle</groupId>
        <artifactId>kaptcha</artifactId>
        <version>2.3.2</version>
    </dependency>
    ```

2. ##### 属性配置
    ```java
    public class KapchaFactory {
        public static DefaultKaptcha getKaptchaBean(){
            Properties properties = new Properties();
            // 验证码是否带边框
            properties.setProperty("kaptcha.border", "no");
            // 验证码字体颜色
            properties.setProperty("kaptcha.textproducer.font.color", "blue");
            // 验证码整体宽度
            properties.setProperty("kaptcha.image.width", "400");
            // 验证码整体高度
            properties.setProperty("kaptcha.image.height", "125");
            // 文字个数
            properties.setProperty("kaptcha.textproducer.char.length", "4");
            // 文字距离
            properties.setProperty("kaptcha.textproducer.char.space", "3");
            // 随机数
            properties.setProperty("kaptcha.textproducer.char.string", "0123456789abcefghijklmnopqrstuvwxyz");
            // 文字大小
            properties.setProperty("kaptcha.textproducer.font.size", "120");
            // 文字随机字体
            properties.setProperty("kaptcha.textproducer.font.names", "宋体,楷体,微软雅黑");
            // 干扰线颜色
            properties.setProperty("kaptcha.noise.color", "blue");
            // 干扰实现类
            properties.setProperty("kaptcha.noise.impl", "com.google.code.kaptcha.impl.DefaultNoise");
            // 图片样式
            properties.setProperty("kaptcha.obscurificator.impl", "com.google.code.kaptcha.impl.WaterRipple");
            Config config = new Config(properties);
            DefaultKaptcha defaultKaptcha = new DefaultKaptcha();
            defaultKaptcha.setConfig(config);
            return defaultKaptcha;
        }
    }
    ```

3. ##### 注入Spring容器
    ```java
    @Configuration
    public class KachaConfiguration {

        @Bean
        DefaultKaptcha getDefaultKaptcha() {
            return KapchaFactory.getKaptchaBean();
        }
    }
    ```

4. ##### 返回验证码
    ```java
    @RestController
    public class Controller {

        private final DefaultKaptcha kaptcha;
        
        @GetMapping("verification-code")
        public void getVerificationCode(HttpServletResponse response) {
            String code = kaptcha.createText();
            Long id = snowflakeIdWorker.nextId();
            redisTemplate.opsForValue().set(CommonConstants.VERIFICATION_CODE_REDIS_KEY + id, code,
                    CommonConstants.VERIFICATION_CODE_EXPIRE_TIME, TimeUnit.SECONDS);
            response.setDateHeader("Expires", 0);
            // Set standard HTTP/1.1 no-cache headers.
            response.setHeader("Cache-Control","no-store, no-cache, must-revalidate");
            // Set IE extended HTTP/1.1 no-cache headers (use addHeader).
            response.addHeader("Cache-Control", "post-check=0, pre-check=0");
            // Set standard HTTP/1.0 no-cache header.
            response.setHeader("Pragma", "no-cache");
            response.setContentType("image/jpeg");
            response.setHeader("id", id.toString());
            BufferedImage image = kaptcha.createImage(code);
            try (ServletOutputStream out = response.getOutputStream()) {
                ImageIO.write(image, "jpg", out);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```