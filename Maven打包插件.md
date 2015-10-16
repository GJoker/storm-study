如果使用Maven进行打包，并且希望将所有的依赖都都放入jar包中，只需将如下代码写入pom.xml文件中。

```
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
      <descriptorRefs>  
        <descriptorRef>jar-with-dependencies</descriptorRef>
      </descriptorRefs>
      <archive>
        <manifest>
          <mainClass>com.path.to.main.Class(主类路径)</mainClass>
        </manifest>
      </archive>
    </configuration>
  </plugin>
```
然后再运行 mvn assembly:assembly 就能得到所需要的jar包了，需要注意不要将 Storm 自带的 jar 放进去，因为集群已经把 Storm 放在了类路径上，否则会报 "NoSuchMethodError"。