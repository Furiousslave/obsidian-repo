##### \<pluginManagement\>
\<pluginManagement\> in parent pom allows to use plugins with same version and configuration in submodules:
```
<project>
    <groupId>com.example</groupId>
    <artifactId>parent-project</artifactId>
    <version>1.0.0</version>
    
    <build>
        <pluginManagement>
            <plugins>
                <!-- Общая конфигурация плагина для всех проектов-потомков -->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.8.1</version>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                </plugin>
                <!-- Другие плагины -->
            </plugins>
        </pluginManagement>
    </build>
    
    <!-- Другие конфигурации и зависимости -->
</project>
```

To use this plugin with such configuration and version in submodule it's necessary to specify it in submodule's pom:
```
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>parent-project</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>child-project</artifactId>
    <version>1.0.0</version>
    
    <build>
        <plugins>
            <!-- Объявление, что мы хотим использовать плагин из <pluginManagement> -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    
    <!-- Другие конфигурации и зависимости -->
</project>
```

##### Inheritance from parent POM
The following elements are inherited form the parent pom:
- dependencies
- plugins
- properties
- repositories
- modules ?
-  build management ?