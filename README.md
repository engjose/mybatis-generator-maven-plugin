# ov-mybatis-generator
基于spring的mybatis-generator生成器

### 背景
SSM 框架在现在互联网企业JAVA选型中已经相当成熟, 随着微服务的架构思想提出,spring-cloud生态也是逐渐火热,但是持久层的框架Mybatis依旧占据重要地位.
在日常开发中持久层我们会借助一些工具来帮忙我们生成, mybatis官方也提供了相当好的工具mybatis-generator, 本ov-mybatis-generator是基于 maven版
本的mybatis-generator进行二次封装

### 对原生mybatis-generator的改动点
1.对xml && dao注释的优化
2.将数据库的注释映射到实体类中

### 环境
spring-boot-2.1.1.RELEASE, mybatis-generator-1.3.5

### 依赖配置
1.创建Spring Maven 项目, 添加Pom依赖(pom.xml)
```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.3.5</version>
</dependency>
```
2. 因为我们创建的只是个jar包依赖, 所以删除spring-boot启动类

3.添加maven打包插件(pom.xml)
```xml
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.0.2</version>
        <configuration>
            <source>1.8</source>
            <target>1.8</target>
            <encoding>UTF-8</encoding>
        </configuration>
    </plugin>

    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>2.2.1</version>
        <executions>
            <execution>
                <id>attach-source</id>
                <goals>
                    <goal>jar</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
</plugins>
```

4.注意: 一定要以jar包的形式引入到使用的项目中, 所以配置deploy到私服, 至于私服怎么搭建这里就不再累赘(pom.xml)
```xml
<distributionManagement>
    <repository>
        <id>ov-release</id>
        <name>Release Repository of ov</name>
        <url>http://localhost:8081/repository/ov-release/</url>
    </repository>
    <snapshotRepository>
        <id>ov-snapshot</id>
        <name>Snapshot Repository of ov</name>
        <url>http://localhost:8081/repository/ov-snapshot/</url>
    </snapshotRepository>
</distributionManagement>
```
### generator.jar 重写注释流程
1.在使用generatorConfig.xml的时候会发现有一个配置注释的标签, 可以指定注释类的类型type=XXXX, 所以我们需要实现一个自定义注释映射实现类
```xml
<commentGenerator />
```
2.自定义注释实现类 extends DefaultCommentGenerator || 实现 CommentGenerator(generator 的注释由该接口控制), 这里就采用实现接口的方式:
CommentGeneratorAdaptor implements CommentGenerator, 然后CommentGeneratorWrapper extends CommentGeneratorAdaptor实现具体的注释映射

3.CommentGeneratorAdaptor实现: 作为适配空实现CommentGenerator

4.CommentGeneratorWrapper实现:
```java
public class CommentGeneratorWrapper extends CommentGeneratorAdaptor {

    private Properties properties;

    public CommentGeneratorWrapper() {
        properties = new Properties();
    }

    @Override
    public void addConfigurationProperties(Properties properties) {
        // 获取自定义的 properties
        this.properties.putAll(properties);
    }

    @Override
    public void addModelClassComment(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        String author = properties.getProperty("author");
        String dateFormat = properties.getProperty("dateFormat", "yyyy-MM-dd");
        SimpleDateFormat dateFormatter = new SimpleDateFormat(dateFormat);

        // 获取表注释
        String remarks = introspectedTable.getRemarks();

        topLevelClass.addJavaDocLine("/**");
        topLevelClass.addJavaDocLine(" * " + remarks);
        topLevelClass.addJavaDocLine(" *");
        topLevelClass.addJavaDocLine(" * @author " + author);
        topLevelClass.addJavaDocLine(" *");
        topLevelClass.addJavaDocLine(" * @date " + dateFormatter.format(new Date()));
        topLevelClass.addJavaDocLine(" */");
    }

    @Override
    public void addFieldComment(Field field, IntrospectedTable introspectedTable, IntrospectedColumn introspectedColumn) {
        // 获取列注释
        String remarks = introspectedColumn.getRemarks();
        field.addJavaDocLine("/** " + remarks + " */");
    }
}
```
5.这时候我们打包发布即刻:
```bash
mvn clean install deploy
```
### 其他项目依赖(使用方)
1.POM引入 ov-mybatis-generator插件
```xml
<dependency>
    <groupId>com.github.ov</groupId>
    <artifactId>generator</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```
2.以maven插件的形式运行, 所以需要引入原生 mybatis-generator插件:(pom.xml)
```xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.5</version>
    <dependencies>
        <dependency>
            <groupId>com.github.ov</groupId>
            <artifactId>generator</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>

        <!--connector driver jar-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.37</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</plugin>
```
3.在source资源文件夹下面创建generatorConfig.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration PUBLIC
        "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--指定特定数据库的jdbc驱动jar包的位置 -->
    <!--<classPathEntry location="mysql-connector-java-5.1.37-bin.jar"/>-->

    <context id="XMBG-01" targetRuntime="MyBatis3">
        <!-- 生成的 Java 文件的编码 -->
        <property name="javaFileEncoding" value="UTF-8"/>
        <!-- 格式化 Java 代码 -->
        <property name="javaFormatter" value="org.mybatis.generator.api.dom.DefaultJavaFormatter"/>
        <!-- 格式化 XML 代码 -->
        <property name="xmlFormatter" value="org.mybatis.generator.api.dom.DefaultXmlFormatter"/>

        <!-- 自定义注释生成器 -->
        <commentGenerator type="com.github.ov.generator.CommentGeneratorWrapper">
            <property name="author" value="JOSE"/>
            <property name="dateFormat" value="yyyy-MM-dd"/>
        </commentGenerator>

        <!-- 配置数据库连接 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://ip:3306/spring_cloud?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true&amp;failOverReadOnly=false&amp;useSSL=false"
                        userId="xxx" password="xxx">
            <property name="useInformationSchema" value="true" />
        </jdbcConnection>

        <!--默认false Java type resolver will always use java.math.BigDecimal if the database column is of type DECIMAL or NUMERIC. -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- 生成实体的位置 -->
        <javaModelGenerator targetPackage="com.github.product.model.po" targetProject="MAVEN">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="false"/>
        </javaModelGenerator>

        <!-- 生成 Mapper 接口的位置 -->
        <sqlMapGenerator targetPackage="com.github.product.dao" targetProject="MAVEN">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!-- 生成 Mapper XML 的位置 -->
        <javaClientGenerator targetPackage="com.github.product.dao" type="XMLMAPPER" targetProject="MAVEN">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- 设置数据库的表名和实体类名 -->
        <table tableName="product_info" domainObjectName="ProductInfo" mapperName="ProductInfoDao"
               enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false">
            <generatedKey column="id" sqlStatement="JDBC" identity="true" />
        </table>
    </context>
</generatorConfiguration>
```
4.至此我们运行generator插件 || mvn命令运行就会生成我们的持久层类, 在target/ 目录下
```bash
mvn mybatis-generator:generate
```
