# 使用CodeQL检测请求转发漏洞

请求转发是服务器内(注意)的一种跳转行为, 它可以保证在浏览器地址栏不发生改变的情况下去访问当前应用的资源, 实现共享用户一次请求中的数据. 请求转发可访问WEB应用根目录下的资源, 如: `WEB-INF`目录下的资源, 当`WEB-INF`目录下存放一些配置文件时, 容易造成敏感信息泄露.

![unsafe-url-forward](/articles/2021/images/unsafe-url-forward.png)

按照MVC架构设计, 用户请求的数据会经过浏览器传到Servlet, 程序在Servlet中进行业务逻辑处理, 然后将处理结果返回给视图. 将处理结果返回给视图这一过程就会经常用到请求转发.

### ***请求转发漏洞***

当应用程序没有对请求转发的资源链接做验证时, 就会出现资源泄露问题. 下面是存在请求转发漏洞的代码.

```java
public void bad(String url, HttpServletRequest request, HttpServletResponse response) {
	String url = request.getParameter("url");
	... //program logic processing
	try {
		request.getRequestDispatcher(url).forward(request, response);
	} catch (ServletException e) {
		e.printStackTrace();
	} catch (IOException e) {
		e.printStackTrace();
	}
}
```

应用程序接收用户请求数据并进行逻辑处理, 然后将处理后的结果做为响应转发到当前应用程序内部资源, 最后将结果呈现给用户. 上面的示例展示的是`Servlet`原始的请求转发, 在很多`MVC`框架中已经对请求转发以及请求重定向做了封装, 例如: `Spring`框架中的使用`ModelAndView`类实现请求转发功能.


### ***防止请求转发漏洞***

上面我们通过示例知道, 真正造成请求转发漏洞的实际是将资源链接交给了用户去控制, 所以我们要防止此类漏洞的产生, 只需要将资源链接做为固定值即可.

### ***使用CodeQL检测请求转发漏洞***

当我们使用CodeQL去检测一个漏洞前, 首先需要明确Source, Sink, 额外的污点跟踪配置以及消毒剂.

- source: 直接远程用户输入、用户可控的数据库数据
- sink: (1) `javax.servlet.ServletRequest.getRequestDispatcher(sink);`(2)`javax.servlet.http.HttpServletRequest.getRequestDispatcher(sink);`(3)`jakarta.servlet.ServletRequest.getRequestDispatcher(sink);`(4)`jakarta.servlet.http.HttpServletRequest.getRequestDispatcher(sink);`(5)`Spring Mapping`方法返回`String`并且`forward`开头,如:"forward:"+url;(6)Spring Mapping方法返回org.springframework.web.servlet.ModelAndView实例.
- sanitizer: (1) 数据流经过拼接操作, 但不是以`/WEB-INF/`开头;(2) Spring Mapping方法经过拼接操作, 但不是以`forward`开头.


编写sink. 每个类我都做了注释, 方便理解.

```ql
UnsafeUrlForward.ql

import java
import DataFlow
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.frameworks.Servlets

/**
 * A concatenate expression using the string `forward:` on the left.
 *
 * E.g: `"forward:" + url`
 */
class ForwardBuilderExpr extends AddExpr {
  ForwardBuilderExpr() {
    this.getLeftOperand().(CompileTimeConstantExpr).getStringValue() = "forward:"
  }
}

/**
 * A call to `StringBuilder.append` or `StringBuffer.append` method, and the parameter value is `"forward:"`.
 *
 * E.g: `StringBuilder.append("forward:")`
 */
class ForwardAppendCall extends MethodAccess {
  ForwardAppendCall() {
    this.getMethod().hasName("append") and
    this.getMethod().getDeclaringType() instanceof StringBuildingType and
    this.getArgument(0).(CompileTimeConstantExpr).getStringValue() = "forward:"
  }
}

abstract class UnsafeUrlForwardSink extends DataFlow::Node { }

/** A Unsafe url forward sink from getRequestDispatcher method. */
class RequestDispatcherSink extends UnsafeUrlForwardSink {
  RequestDispatcherSink() {
    exists(MethodAccess ma |
      ma.getMethod() instanceof ServletRequestGetRequestDispatcherMethod and
      ma.getArgument(0) = this.asExpr()
    )
  }
}

/** A Unsafe url forward sink from spring controller method. */
class SpringUrlForwardSink extends UnsafeUrlForwardSink {
  SpringUrlForwardSink() {
    exists(ForwardBuilderExpr rbe |
      rbe.getRightOperand() = this.asExpr() and
      any(SpringRequestMappingMethod sqmm).polyCalls*(this.getEnclosingCallable())
    )
    or
    exists(MethodAccess ma, ForwardAppendCall rac |
      DataFlow2::localExprFlow(rac.getQualifier(), ma.getQualifier()) and
      ma.getMethod().hasName("append") and
      ma.getArgument(0) = this.asExpr() and
      any(SpringRequestMappingMethod sqmm).polyCalls*(this.getEnclosingCallable())
    )
    or
    exists(ClassInstanceExpr cie |
      cie.getConstructedType().hasQualifiedName("org.springframework.web.servlet", "ModelAndView") and
      (
        exists(ForwardBuilderExpr rbe |
          rbe = cie.getArgument(0) and rbe.getRightOperand() = this.asExpr()
        )
        or
        cie.getArgument(0) = this.asExpr()
      )
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().hasName("setViewName") and
      ma.getMethod()
          .getDeclaringType()
          .hasQualifiedName("org.springframework.web.servlet", "ModelAndView") and
      ma.getArgument(0) = this.asExpr()
    )
  }
}

java/ql/src/semmle/code/java/frameworks/Servlets.qll

/**
 * The method `getRequestDispatcher(String)` declared in `javax.servlet.http.HttpServletRequest` or `javax.servlet.ServletRequest`.
 */
class ServletRequestGetRequestDispatcherMethod extends Method {
  ServletRequestGetRequestDispatcherMethod() {
    getDeclaringType() instanceof ServletRequest and
    hasName("getRequestDispatcher") and
    getNumberOfParameters() = 1 and
    getParameter(0).getType() instanceof TypeString
  }
}
```

编写sanitizer

```ql
import java
import DataFlow

class UnsafeUrlForwardSanitizer extends DataFlow::Node {
  UnsafeUrlForwardSanitizer() {
    this.getType() instanceof BoxedType
    or
    this.getType() instanceof PrimitiveType
    or
    exists(AddExpr ae |
      ae.getRightOperand() = this.asExpr() and
      (
        not ae.getLeftOperand().(CompileTimeConstantExpr).getStringValue().matches("/WEB-INF/%")
        and
        not ae.getLeftOperand().(CompileTimeConstantExpr).getStringValue() = "forward:"
      )
    )
  }
}
```

### ***写在后面***

使用`CodeQL`检测请求转发漏洞的`sink`和`sanitizer`还需要不断调优, 不断降低漏报率和误报率. 另外, 可以使用`CodeQL`分析一些0day, 继而挖掘一些没有发现的漏洞是.


### ***参考***

[File Disclosure](https://vulncat.fortify.com/en/detail?id=desc.dataflow.java.file_disclosure_spring)
