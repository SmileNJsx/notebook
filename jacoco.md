# Jacoco统计在线程序代码覆盖率

## 前言

本教程所有案例都是以aimms组件源码为素材

## 知识点

+ Dump 镜像 内存快照
+ bat/sh 语法知识
+ tomcat 中文乱码问题

## 环境准备 (物料放在同一文件加下)

+ _aimms.rar_ (公司文件请勿外传)
+ _apache-ant-1.10.5-bin.zip_ (用来实时获取远程dump文件，解析dump文件生成report)
+ _apache-tomcat-9.0.6.rar_ (tomcat运行程序)
+ _jacocoagent.jar_ (tomcat启动，Javaagent程序)
+ _jacocoant.jar_ (ant依赖包，置于ant home lib)
+ _build.xml_ (ant程序，用于实时获取dump，解析dump)
+ _html_ (Jacoco生成代码覆盖率文档)

## 配置参数

### 配置ant环境

#### ant安装配置

_自行安装配置ant_

#### 配置实时监控，build.xml文件

_请根据自身配置修正build.xml_

```xml
<?xml version="1.0" ?>
<project name="coverage" xmlns:jacoco="antlib:org.jacoco.ant">

    <!--ant环境执行依赖包-->
    <property name="jacocoantPath" value="D:/apache-ant-1.10.5-bin/apache-ant-1.10.5/lib/jacocoant.jar"/>
    <!--dump文件存储，指定到文件-->
    <property name="jacocoexecPath" value="D:/188/jacoco.exec"/>
    <!--html报告生成路径-->
    <property name="reportfolderPath" value="D:/189"/>
    <!--需要解析的源码地址-->
    <property name="webSrcpath" value="D:/1903/aimms3/aimms-business/src/main/java/" />
    <!--需要解析的class地址-->
    <property name="webClasspath" value="D:/1903/aimms3/aimms-business/target/classes/" />
    <!--ant task def-->
    <taskdef uri="antlib:org.jacoco.ant" resource="org/jacoco/ant/antlib.xml">
        <classpath path="D:/apache-ant-1.10.5-bin/apache-ant-1.10.5/lib/jacocoant.jar"></classpath>
    </taskdef>
    <!--远程实时dump地址配置-->
    <target name="dump">
        <jacoco:dump address="127.0.0.1" port="8081" reset="false" destfile="${jacocoexecPath}" append="false"></jacoco:dump>
    </target>
    <!--html report生产配置-->
    <target name="report">
        <delete dir="${reportfolderPath}" />
        <mkdir dir="${reportfolderPath}" />
    <jacoco:report>
        <executiondata>
        <file file="${jacocoexecPath}" />
         </executiondata>

        <structure name="JaCoCo Report">
            <!--需要解析文件配置 可多个group-->
            <group name="Launch related">
                <classfiles>
                    <fileset dir="${webClasspath}" />
                </classfiles>
                <sourcefiles encoding="gbk">
                    <fileset dir="${webSrcpath}" />
                </sourcefiles>
            </group>
        </structure>
        <html destdir="${reportfolderPath}" encoding="utf-8" />
    </jacoco:report>
    </target>

</project>
```

### 配置Tomcat环境

#### Tomcat安装配置

_自行安装配置Tomcat_

#### 配置Tomcat catalina.bat/sh，远程实时dump

```bat
setlocal --在原文此处下加一行
set "JAVA_OPTS=%JAVA_OPTS% -javaagent:D:\188\jacocoagent.jar=includes=*,output=tcpserver,address=127.0.0.1,port=8081"
```

### idea开发工具

_待完善_

### 生产环境 （教程步骤）

_默认环境都已经安装完善_

+ 解压导入aimms源码到IDE中，根目录下新建build.xml，内容如上 _(需要maven，自行安装设置)_。
+ 打包aimms，可用mvn clean package，将生成的war包放到Tomcat webapps下。
+ 修改Tomcat catalina.bat/sh，如上教程；启动Tomcat，请使用startup.bat/sh。
+ aimms根目录下运行 ant dump可实时导出远程Tomcat的dump。
+ aimms根目录下运行 ant report可生成html解析文档。
+ 详情见 自动化测试tomcat.docx

## 注意事项

_待完善_

## 后续研究 _(尚未解决问题)_

+ 目前只支持在线运行dump数据模型，Tomcat关闭dump数据依然无法与class和源码对应解析
+ 目前对dump数据的解析采用的是ant解析的方式，依然无法采用jacoco源码core和report包代码去编码解析
+ 目前依然无法完成直接在idea工具中的dump实时分析
