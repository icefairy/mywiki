# Maven 使用

同时引入一个包的多个版本方法
```xml
<dependency>

    <groupId>cn.hutool</groupId>

    <artifactId>hutool-core</artifactId>

</dependency>

<!--添加同一依赖的不同版本：groupId后加.-->

<dependency>

    <groupId>cn.hutool.</groupId>

    <artifactId>hutool-core</artifactId>

    <version>4.1.12</version>

</dependency>

<!--添加同一依赖的不同版本：groupId后加..-->

<dependency>

    <groupId>cn.hutool..</groupId>

    <artifactId>hutool-core</artifactId>

    <version>4.5.1</version>

</dependency>
```
```bash
#复制依赖到lib文件夹
mvn dependency:copy-dependencies -DoutputDirectory=target\lib -DincludeScope=compile
#一般结合以下配置使用

```

```xml
<build>
    <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.6</version>
        <configuration>
            <archive>
                <manifest>
                    <classpathPrefix>lib/</classpathPrefix>
                    <addClasspath>true</addClasspath>
                    <mainClass>${main.class}</mainClass>
                </manifest>
            </archive>
        </configuration>
        </plugin>
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <outputDirectory>${project.build.directory}/lib</outputDirectory>
                </configuration>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <overWriteReleases>false</overWriteReleases>
                            <overWriteSnapshots>false</overWriteSnapshots>
                            <overWriteIfNewer>true</overWriteIfNewer>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
    </plugins>
</build>
```

