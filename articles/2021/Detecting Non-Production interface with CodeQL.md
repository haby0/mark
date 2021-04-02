# Detecting Non-Production interface with CodeQL

## **Description**

Have we encountered this situation when writing programs? For the convenience of testing, directly write the test or debugging interface to the production system directory, and when packaging the program to the production system, the test interface is not excluded, resulting in the test interface Left on the production system. At this time, the production system is extremely insecure. Next, I will introduce in detail how to use `CodeQL` to detect such problems, so that problems can be found in time before the program goes online, and more losses can be avoided.

## **The hazards of non-production interfaces**

Non-production interfaces are insecure by default, because attackers can use test or debugging interfaces accidentally enabled on the production system to collect information or take advantage of some functions that are otherwise unavailable. For details about the defect, please refer to [CAPEC-121: Exploit Non-Production Interfaces](https://capec.mitre.org/data/definitions/121.html).

### ***Example***

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

The above sample code shows that there is a test interface class `TestApiController` in the production directory of `com.production.controller`. The user can directly access the intranet by visiting `http://xxx.com/testRequest?url=bad.com` system message.

### ***How to fix***

There are two common methods:

1. The more violent method is to directly comment out the test interface class or test interface method
2. Use the packaging tool to exclude relevant test interface class files when packaging, such as `Maven`/`Gradle`/`Ant`, etc.

## **Detecting Non-Production interface with CodeQL**

[CodeQL](https://securitylab.github.com/tools/codeql) is an industry-leading semantic code analysis engine, we can use it to find vulnerabilities in the entire code base. In addition, it is also a free research and open source software.


We write two ql, respectively:

- Use web.xml to define the access interface
- Use SpringController to define access interface

## ***CodeQL detection uses web.xml to define non-production access interfaces***

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

web.xml is shown above. The general idea is:

- Find the `url-pattern` under the `servlet-mapping` tag
- Locate the `servlet-class` under the `servlet` tag, and make an equivalent judgment on the two `servlet-name`
- Verify whether the `servlet-class` under the `servlet` tag is packaged in `pom.xml` and is excluded


So the final ql is

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

## ***CodeQL detection uses SpringController to define non-production interfaces***

The general idea is:

- Find the method annotated with `Mapping` or the class annotated with `WebServlet`
- Check the class found in the previous step, and see if the jar or war was excluded when `pom.xml` was packaged


So the final ql is:

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

## **Reference**

[CAPEC-121: Exploit Non-Production Interfaces](https://capec.mitre.org/data/definitions/121.html)