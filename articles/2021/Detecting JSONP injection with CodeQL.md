# Detecting JSONP injection with CodeQL

## **Description**

`JSONP` injection is a very common and extremely dangerous vulnerability. It is very common because many websites need to transmit data across domains. It is extremely dangerous because many of the data transmitted are sensitive information. Attackers can be without the user's knowledge Under circumstances, to obtain user data, if combined with worm vulnerabilities, the harm will be unimaginable. Therefore, we should try to avoid such vulnerabilities when designing programs to protect our users. Next, I will introduce in detail how to use `CodeQL` to detect `JSONP` Injection (to be precise, this article is only partially tested), because the formation of `JSONP` injection vulnerabilities also depends on other conditions, such as: whether or not the origin is tested. But its detection is still necessary.

## **Harm of JSONP injection**

The main harm that `JSONP` injection can cause is website or user data leakage. The following gives an example and describes how to fix the vulnerability.


### ***Example***

```java
package com.production.controller

import com.google.gson.Gson;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class TestApiController {

    private static HashMap hashMap = new HashMap();

    static {
        hashMap.put("username","admin");
        hashMap.put("password","123456");
    }

    @GetMapping(value = "jsonp")
    public void bad1(HttpServletRequest request, HttpServletResponse response) throws Exception {
        String resultStr = null;
        String jsonpCallback = request.getParameter("jsonpCallback");
        Gson gson = new Gson();
        String result = gson.toJson(hashMap);
        resultStr = jsonpCallback + "(" + result + ")";
        return resultStr;
    }
}
```

The above code example is a standard JSONP data transmission. The parameter value of `jsonpCallback` passed by the user is received as the function name of the JSONP data, and the account password is used as the function body, and then spliced and returned to the user. If it is an attacker, through the control` jsonpCallback` parameter value, and then use cross-domain issues to obtain the account password of the website user.

### ***How to fix***

How can we fix this problem? Here are a few suggestions:

1. One point of violence is to completely delete the JSONP function
2. Add the Access-Control-Allow-Origin header to your API response
3. Check for Referer

## **Detecting JSONP injection with CodeQL**

I used [CodeQL](https://securitylab.github.com/tools/codeql) to do security analysis on some open source projects, it is great, and some security issues have been unearthed, and they really exist. It is recommended to use [ lgtm](https://lgtm.com/), the analyzed security issues will show all the paths tracked by the taint, which is very convenient.

Next, we write our ql. In fact, it is to convert the principle of vulnerability formation into ql statements.

- JSONP injection can only be a GET request, and there is no request body
- Compliant with JSONP format data
- Function name user controllable
- Return as a response


`JSONP` injection can only be a GET request, and there is no request body. We can implement the following ql statement.

```ql
import java
import DataFlow
import semmle.code.java.frameworks.spring.SpringController

/**
 * A method that is called to handle an HTTP GET request.
 */
abstract class RequestGetMethod extends Method {
  RequestGetMethod() {
    not exists(MethodAccess ma |
      ma.getMethod() instanceof ServletRequestGetBodyMethod and
      any(this).polyCalls*(ma.getEnclosingCallable())
    )
  }
}

/** Override method of `doGet` of `Servlet` subclass. */
private class ServletGetMethod extends RequestGetMethod {
  ServletGetMethod() { this instanceof DoGetServletMethod }
}

/** The method of SpringController class processing `get` request. */
abstract class SpringControllerGetMethod extends RequestGetMethod { }

/** Method using `GetMapping` annotation in SpringController class. */
class SpringControllerGetMappingGetMethod extends SpringControllerGetMethod {
  SpringControllerGetMappingGetMethod() {
    this.getAnAnnotation()
        .getType()
        .hasQualifiedName("org.springframework.web.bind.annotation", "GetMapping")
  }
}

/** The method that uses the `RequestMapping` annotation in the SpringController class and only handles the get request. */
class SpringControllerRequestMappingGetMethod extends SpringControllerGetMethod {
  SpringControllerRequestMappingGetMethod() {
    this.getAnAnnotation()
        .getType()
        .hasQualifiedName("org.springframework.web.bind.annotation", "RequestMapping") and
    (
      this.getAnAnnotation().getValue("method").(VarAccess).getVariable().getName() = "GET" or
      this.getAnAnnotation().getValue("method").(ArrayInit).getSize() = 0 //Java code example: @RequestMapping(value = "test")
    ) and
    not this.getAParamType().getName() = "MultipartFile"
  }
}

```

The `RequestGetMethod` abstract class excludes the method processing the request body, which is universal. Then through the `ServletGetMethod` and `SpringControllerGetMethod` implementation classes, the get request method of the `Servlet` implementation class and the get request of the `SpringController` are respectively defined Method. When using `this.getAnAnnotation().getValue("method")` to limit, I originally used `toString()`, with the help of [smowton](https://github.com/smowton) , Modified to the current query, which is more concise and beautiful.

In line with JSONP format data, we can implement the ql statement.

```ql
import java
import semmle.code.java.dataflow.DataFlow

/**
 * A concatenate expression using `(` and `)` or `);`.
 *
 * E.g: `functionName + "(" + json + ")"` or `functionName + "(" + json + ");"`
 */
class JsonpBuilderExpr extends AddExpr {
  JsonpBuilderExpr() {
    getRightOperand().(CompileTimeConstantExpr).getStringValue().regexpMatch("\\);?") and
    getLeftOperand()
        .(AddExpr)
        .getLeftOperand()
        .(AddExpr)
        .getRightOperand()
        .(CompileTimeConstantExpr)
        .getStringValue() = "("
  }

  /** Get the jsonp function name of this expression. */
  Expr getFunctionName() {
    result = getLeftOperand().(AddExpr).getLeftOperand().(AddExpr).getLeftOperand()
  }

  /** Get the json data of this expression. */
  Expr getJsonExpr() { result = getLeftOperand().(AddExpr).getRightOperand() }
}


/** Json string type data. */
abstract class JsonStringSource extends DataFlow::Node { }

/**
 * Convert to String using Gson library. *
 *
 * For example, in the method access `Gson.toJson(...)`,
 * the `Object` type data is converted to the `String` type data.
 */
private class GsonString extends JsonStringSource {
  GsonString() {
    exists(MethodAccess ma, Method m | ma.getMethod() = m |
      m.hasName("toJson") and
      m.getDeclaringType().getASupertype*().hasQualifiedName("com.google.gson", "Gson") and
      this.asExpr() = ma
    )
  }
}

/**
 * Convert to String using Fastjson library.
 *
 * For example, in the method access `JSON.toJSONString(...)`,
 * the `Object` type data is converted to the `String` type data.
 */
private class FastjsonString extends JsonStringSource {
  FastjsonString() {
    exists(MethodAccess ma, Method m | ma.getMethod() = m |
      m.hasName("toJSONString") and
      m.getDeclaringType().getASupertype*().hasQualifiedName("com.alibaba.fastjson", "JSON") and
      this.asExpr() = ma
    )
  }
}

/**
 * Convert to String using Jackson library.
 *
 * For example, in the method access `ObjectMapper.writeValueAsString(...)`,
 * the `Object` type data is converted to the `String` type data.
 */
private class JacksonString extends JsonStringSource {
  JacksonString() {
    exists(MethodAccess ma, Method m | ma.getMethod() = m |
      m.hasName("writeValueAsString") and
      m.getDeclaringType()
          .getASupertype*()
          .hasQualifiedName("com.fasterxml.jackson.databind", "ObjectMapper") and
      this.asExpr() = ma
    )
  }
}
```

Finally, the function name can be controlled by the user as a `source`, using `RemoteFlowSource`. As a response return can be used as a `sink`, using `XssSink`. Note: (1) How to make our `source` After the JSONP format data, and as the function name, we need to implement a new taint tracking configuration; (2) The function body of the JSONP format data also needs to implement the taint tracking configuration. The final `ql` statement can be viewed [lgtm result](https://lgtm.com/query/902990529688935174/).

## **Reference**

[Practical JSONP Injection](https://securitycafe.ro/2017/01/18/practical-jsonp-injection)