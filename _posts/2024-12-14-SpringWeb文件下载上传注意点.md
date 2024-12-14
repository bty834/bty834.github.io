---
title: Spring Web文件上传下载注意点
categories: [ 编程, Spring ]
tags: [ spring ]
---

我们使用如下的Controller来上传下载文件：
```java
@RestController
@RequestMapping("/hello")
@Slf4j
public class FileController {

    @GetMapping("/upload")
    public void uploadFile(@RequestParam("file") MultipartFile file) {
        log.info("{} uploaded, {} bytes", file.getOriginalFilename(), file.getSize());
    }

    @GetMapping("/download")
    public void downloadFile(HttpServletResponse response) {
        List<TestVO> voList = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            TestVO vo = new TestVO();
            vo.setId(String.valueOf(i));
            vo.setName(i + "name");
            voList.add(vo);
        }
        try {
            response.setCharacterEncoding(StandardCharsets.UTF_8.name());
            response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);
            response.setHeader(HttpHeaders.CONTENT_DISPOSITION, String.format("attachment; filename=%s", "test.xlsx"));
            EasyExcel.write(response.getOutputStream()).sheet(0).doWrite(voList);
            // OutputStream outputStream = response.getOutputStream();
            // easyexcel会flush，这里不用
            // outputStream.flush();
            log.info("response: {}", JsonUtil.toJson(response));
        } catch (IOException e) {
            log.error("下载失败：e:", e);
            throw new RuntimeException("文件下载失败");
        }
    }
}
```

## Upload
当我们调用upload接口分别上传一个15MB和8MB的文件时会出现以下两个报错：

```txt
// 上传15MB文件时
org.apache.tomcat.util.http.fileupload.impl.SizeLimitExceededException: the request was rejected because its size (15579467) exceeds the configured maximum (10485760)

// 上传8MB的文件时
org.apache.tomcat.util.http.fileupload.impl.FileSizeLimitExceededException: The field file exceeds its maximum permitted size of 1048576 bytes.
```
原来SpringBoot会对tomcat上传form/multipart的大小做限制，multipart可一次上传多个文件，
默认每个文件大小不超过1MB，文件总大小不超过10MB。参见如下默认配置：

```java
@ConfigurationProperties(prefix = "spring.servlet.multipart", ignoreUnknownFields = false)
public class MultipartProperties {

  // 默认开启
	private boolean enabled = true;
  
  // 默认1MB
	private DataSize maxFileSize = DataSize.ofMegabytes(1);

  // 默认10MB
	private DataSize maxRequestSize = DataSize.ofMegabytes(10);
  ...
}
```
可通过配置文件设置修改默认配置:
```properties
spring.servlet.multipart.max-request-size=200MB
spring.servlet.multipart.max-file-size=200MB
```
也可注入自定义配置类Bean:
```java
    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        factory.setMaxFileSize(DataSize.ofMegabytes(200));
        factory.setMaxRequestSize(DataSize.ofMegabytes(200));
        return factory.createMultipartConfig();
    }
```
但是上面的配置方式无法动态变化，只能修改代码重新上线。此时可继承MultipartConfigElement重写get方法配合配置中心达到动态配置的目的，代码如下：

```java
@Component
public class DynamicMultipartConfigElement extends MultipartConfigElement {
    public DynamicMultipartConfigElement() {
        super("");
    }

    @Override
    public long getMaxFileSize() {
        // 从配置中心获取最新配置
        return super.getMaxFileSize();
    }

    @Override
    public long getMaxRequestSize() {
        // // 从配置中心获取最新配置
        return super.getMaxRequestSize();
    }
}
```
## Download

运行以上下载文件的代码，发现会报错：

```txt
com.fasterxml.jackson.databind.JsonMappingException: getOutputStream() has already been called for this response (through reference chain: org.apache.catalina.connector.ResponseFacade["writer"])
```
看提示发现是在outputSteam#flush完再次通过打印日志的JsonUtil.toJson操作调用了outputStream，而stream只能被消费一次。
通常打印日志是通过AOP注解实现，使用时不会关注注解切面的内部逻辑，如果在切面执行完再调用打印入参的response则会报此错。
当然，打印日志即使报错也不该影响正常的业务逻辑，所以在日志AOP切面处理时要对打日志的逻辑加上try-catch
