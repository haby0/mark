# 使用CodeQL检测非生产接口

## **描述**

我们在写程序时有没有遇到过这种情况,为了测试的便利,直接将测试或者调试接口写到生产系统目录下,而在将程序打包到生产系统由于没有排除掉测试接口,导致测试接口遗留在生产系统上.此时的生产系统存在极大的不安全性,接下来我详细介绍如何使用CodeQL去检测此类问题,这样能在程序上线前及时发现问题,避免更多的损失.

## **非生产接口的危害**

非生产接口默认情况下是不安全的,因为攻击者利用在生产系统上意外启用的测试或调试接口,可以收集信息或者利用一些原本无法获取的功能.该缺陷详细介绍请参见 [CAPEC-121: Exploit Non-Production Interfaces](https://capec.mitre.org/data/definitions/121.html).

### ***示例***

```java
package com.production.controller

import java.net.URL;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class TestApiController {

    @GetMapping(value = "testRequest")
    public void bad3(HttpServletRequest request, HttpServletResponse response) throws Exception {
        URL url = new URL(request.getParameter("url"));
        url.openStream();
    }
}
```

上面示例代码展示了在`com.production.controller`生产目录下存在测试接口类`TestApiController`.用户通过访问`http://xxx.com/testRequest?url=bad.com`,可以直接访问内网系统信息.

### ***如何修复***

常见的有两种方法:

1. 比较暴力的方法是直接将测试接口类或者测试接口方法注释掉
2. 利用打包工具在打包时排除掉相关测试接口类文件,如`Maven`/`Gradle`/`Ant`等

## **使用CodeQL检测非生产接口**

[CodeQL](https://securitylab.github.com/tools/codeql)是行业领先的语义代码分析引擎,我们可以使用它发现整个代码库中的漏洞.另外,它还是一个免费的研究和开源软件.

我们编写两条ql,分别是

- 使用web.xml定义访问接口
- 使用SpringController定义访问接口

## ***CodeQL检测使用web.xml定义的非生产接口***

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>

  <servlet>
    <servlet-name>testServlet</servlet-name>
    <servlet-class>TestServlet</servlet-class>
  </servlet>

  <servlet-mapping>
    <servlet-name>testServlet</servlet-name>
    <url-pattern>/testServlet</url-pattern>
  </servlet-mapping>
</web-app>
```

web.xml如上所示. 大致思路是

- 找到`servlet-mapping`标签下的`url-pattern`
- 定位`servlet`标签下的`servlet-class`,对两个`servlet-name`做等值判断
- 验证`servlet`标签下的`servlet-class`是否在`pom.xml`打包是被排除掉


所以最终的ql是

```ql
/**
 * @name Servlet Non Production Interface
 * @description Servlet non production interface is left behind and vulnerable to attacks.
 * @kind problem
 * @id java/servlet-non-production-interface
 * @tags security
 *       external/cwe/cwe-749
 */

import java
import semmle.code.xml.WebXML
import semmle.code.xml.MavenPom
import semmle.code.java.frameworks.Servlets

/** Verify that the class is `pom.xml` to build the war package exclusion class. */
predicate verifyClassWithPomWarExcludes(Class c) {
  not exists(PackagingExcludes pe |
    pe.getAnValue() =
      "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + c.getName() + ".class" or
    pe.getAnValue() =
      "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/" + c.getName() +
        ".class" or
    pe.getAnValue() = "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/*" or
    pe.getAnValue() = "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/**"
  )
}

/**
 * A `<servlet-mapping>` element in a `web.xml` file.
 */
class WebServletMapping extends WebXMLElement {
  WebServletMapping() { getName() = "servlet-mapping" }
}

/**
 * A `<url-pattern>` element in a `web.xml` file, nested under a `<servlet-mapping>` element.
 */
class WebServletMappingUrlPattern extends WebXMLElement {
  WebServletMappingUrlPattern() {
    getName() = "url-pattern" and
    getParent() instanceof WebServletMapping
  }
}

from WebServletMappingUrlPattern wsmup, WebServletClass wsc
where
  wsmup.getValue().regexpMatch("(?i).*test.*") and // The test interface is defined in `web.xml`
  wsc.getParent().getAChild("servlet-name").getTextValue() =
    wsmup.getParent().getAChild("servlet-name").getTextValue() and
  verifyClassWithPomWarExcludes(wsc.getClass())
select wsmup, "servlet non production interface is left behind and vulnerable to attacks"
```

## ***CodeQL检测使用SpringController定义的非生产接口***

大致思路是

- 找到使用`Mapping`注解的方法或者`WebServlet`注解的类
- 对上一步找到的类做校验,看`pom.xml`打包jar或者war时有没有做排除


所以最终的ql是

```ql
/**
 * @name Non Production Interface
 * @description Non-Production interface is left behind and vulnerable to attacks.
 * @kind problem
 * @id java/non-production-interface
 * @tags security
 *       external/cwe/cwe-749
 */

import java
import semmle.code.xml.MavenPom
import semmle.code.java.frameworks.spring.SpringController


/** Verify that the class is `pom.xml` to build the jar package exclusion class. */
predicate verifyClassWithPomJarExcludes(Class c) {
  not exists(Exclude e |
    e.getValue() = c.getPackage().getName().replaceAll(".", "/") + "/*" or
    e.getValue() = c.getPackage().getName().replaceAll(".", "/") + "/**" or
    e.getValue() = c.getPackage().getName().replaceAll(".", "/") + c.getName() + ".class" or
    e.getValue() = c.getPackage().getName().replaceAll(".", "/") + "/" + c.getName() + ".class"
  )
}

/** Verify that the class is `pom.xml` to build the war package exclusion class. */
predicate verifyClassWithPomWarExcludes(Class c) {
  not exists(PackagingExcludes pe |
    pe.getAnValue() =
      "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + c.getName() + ".class" or
    pe.getAnValue() =
      "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/" + c.getName() +
        ".class" or
    pe.getAnValue() = "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/*" or
    pe.getAnValue() = "WEB-INF/classes/" + c.getPackage().getName().replaceAll(".", "/") + "/**"
  )
}


abstract class NonProductionInterface extends Annotation {
  NonProductionInterface() {
    not exists(this.getLocation().getFile().getAbsolutePath().indexOf("/src/test/java"))
  }
}

/** Spring Controller non-production interface. */
class SpringControllerNonProductionInterface extends NonProductionInterface {
  SpringControllerNonProductionInterface() {
    exists(SpringRequestMappingMethod srmm |
      srmm.getAnAnnotation() = this and
      srmm.getNumberOfLinesOfCode() > 2 and  //Ensure that the test interface method has a body.
      this.getType() instanceof SpringRequestMappingAnnotationType and
      this.getAValue("value").toString().regexpMatch("(?i).*test.*")
    )
  }
}

/** An annotation type that identifies webservlet. */
class WebServletAnnotation extends AnnotationType {
  WebServletAnnotation() {
    // `@WebServlet` used directly as an annotation.
    hasQualifiedName("javax.servlet.annotation", "WebServlet")
    or
    // `@WebServlet` can be used as a meta-annotation on other annotation types.
    getAnAnnotation().getType() instanceof WebServletAnnotation
  }
}

/** Use the `servlet` subclass of the `@WebServlet` interface. */
class UseWebServletInterfaceClass extends Class {
  UseWebServletInterfaceClass() {
    getAnAnnotation().getType() instanceof WebServletAnnotation and
    getASupertype*().hasQualifiedName("javax.servlet", "Servlet")
  }
}

/** WebServlet non-production interface. */
class WebServletNonProductionInterface extends NonProductionInterface {
  WebServletNonProductionInterface() {
    exists(UseWebServletInterfaceClass wsc |
      wsc.getAnAnnotation() = this and
      this.getAValue("value").toString().regexpMatch("(?i).*test.*")
      or
      this.getAValue("urlPatterns").toString().regexpMatch("(?i).*test.*")
    )
  }
}

from Class c, NonProductionInterface npi
where
  npi.getCompilationUnit() = c.getCompilationUnit() and
  verifyClassWithPomJarExcludes(c) and
  verifyClassWithPomWarExcludes(c)
select npi, "non production interface is left behind and vulnerable to attacks"
```

## **参考**

[CAPEC-121: Exploit Non-Production Interfaces](https://capec.mitre.org/data/definitions/121.html)