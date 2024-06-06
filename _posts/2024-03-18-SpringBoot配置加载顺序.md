---
title: SpringBoot配置加载顺序
categories: [编程,Java]
tags: [springboot]
---

[Externalized Configuration](https://docs.spring.io/spring-boot/docs/1.5.22.RELEASE/reference/html/boot-features-external-config.html)
优先级，序号越小优先级越高，优先级为1的会覆盖优先级为17的
1. Devtools global settings properties on your home directory (~/.spring-boot-devtools.properties when devtools is active).
2. `@TestPropertySource` annotations on your tests.
3. `@SpringBootTest#properties` annotation attribute on your tests.
4. Command line arguments.
5. Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property)
6. `ServletConfig` init parameters.
7. `ServletContext` init parameters.
8. JNDI attributes from java:comp/env.
9. Java System properties (System.getProperties()). 通过 `java -jar -Dkey=val ...`设置的
10. OS environment variables.
11. A RandomValuePropertySource that only has properties in random.*.
12. Profile-specific application properties outside of your packaged jar (application-{profile}.properties and YAML variants) yml->yaml->properties （后面会覆盖前面的配置）
13. Profile-specific application properties packaged inside your jar (application-{profile}.properties and YAML variants)
14. Application properties outside of your packaged jar (application.properties and YAML variants).
15. Application properties packaged inside your jar (application.properties and YAML variants).
16. `@PropertySource` annotations on your `@Configuration` classes.
17. Default properties (specified using SpringApplication.setDefaultProperties).
