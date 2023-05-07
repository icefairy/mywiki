# JPA分页最大条数限制修改

jpa源码内部有个最大条数2000条的限制，可以在配置文件中进行修改；
```yaml
spring:
  data:
    web:
      pageable:
        # 默认页面大小
        default-page-size: 20
        # 接受的最大页面大小
        max-page-size: 20000
```