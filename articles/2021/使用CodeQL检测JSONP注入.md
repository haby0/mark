# 使用CodeQL检测JSONP注入

## **描述**

JSONP注入是一个非常普遍而且危害性极高的漏洞.非常普遍是因为很多网站都需要跨域传输数据.危害性极高是因为传输的数据很多都是敏感信息.攻击者可以在用户不知情的情况下获取用户的数据,如果结合蠕虫漏洞,危害性会难以想象.所以,我们应该在设计程序时,尽量避免此类漏洞,保护我们的用户.接下来我会详细介绍如何使用CodeQL去检测JSONP注入(准确的说,本文只是做了部分检测),因为JSONP注入漏洞的形成还依赖别的条件,比如:有没有对进行origin检测.但是它的检测任然是有必要的.

## **JSONP注入的危害**

JSONP注入主要能够造成的危害就是网站或用户数据泄露.下文给出了示例以及介绍了该漏洞如何进行修复.


### ***示例***

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

上面的代码示例是标准的JSONP数据传输,接收用户传入的`jsonpCallback`参数值做为JSONP数据的函数名,账号密码做为函数体,然后拼接返回给用户.如果是攻击者,通过控制`jsonpCallback`参数值,然后利用跨域的问题来获取网站用户的账号密码.

### ***如何修复***

我们如何修复该问题呢,在这里提供了几种建议:

1. 暴力一点就是完全删除JSONP功能
2. 将Access-Control-Allow-Origin标头添加到您的API响应中
3. 针对Referer做校验

## **使用CodeQL检测JSONP注入**

我使用[CodeQL](https://securitylab.github.com/tools/codeql)对一些开源项目做过安全分析,它是很棒的,目前已经挖掘出一些安全问题,并且真实存在.推荐使用[lgtm](https://lgtm.com/),分析出的安全问题会展示污点跟踪的所有路径,很方便.

接下来编写我们的ql.其实就是将漏洞形成原理转换成ql语句.

- JSONP注入只能是GET请求,并且没有请求体
- 符合JSONP格式数据
- 函数名用户可控
- 做为响应返回


JSONP注入只能是GET请求,并且没有请求体.我们可以实现以下ql语句.

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

`RequestGetMethod`抽象类排除了方法处理请求体的情况,这是通用的.然后通过`ServletGetMethod`和`SpringControllerGetMethod`实现类,分别定义了`Servlet`实现类的get请求方法和`SpringController`的get请求方法.在使用`this.getAnAnnotation().getValue("method")`做限制时,我原本使用的是`toString()`,在[smowton](https://github.com/smowton)帮助下,修改为现在的查询,更佳简洁美观.

符合JSONP格式数据,我们可以实现一下ql语句

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

最后函数名用户可控可以做为的`source`,使用`RemoteFlowSource`.做为响应返回可以做为`sink`,使用`XssSink`.需要注意的是:(1)如何让我们的`source`经过JSONP格式数据,并且做为函数名,我们需要实现新的污点跟踪配置;(2)JSONP格式数据的函数体也需要实现污点跟踪配置.


最终的`ql`语句可以查看[lgtm result](https://lgtm.com/query/902990529688935174/)

## **参考**

[Practical JSONP Injection](https://securitycafe.ro/2017/01/18/practical-jsonp-injection)