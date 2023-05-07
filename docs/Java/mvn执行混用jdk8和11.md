例如执行sonar的任务，先设置为jdk8执行项目编译（假设项目必须依赖jdk8），编译完成后再切成jdk11执行sonar的任务即可。
```bat
set JAVA_HOME=d:\soft\jdk8
mvn clean verify -DskipTests=true
set JAVA_HOME=d:\soft\jdk11
mvn sonar:sonar   -Dsonar.projectKey=zhxq   -Dsonar.host.url=http://ip:9000   -Dsonar.login=apitoken -DskipTests=true
```