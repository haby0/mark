# Detecting Request forwarde vulnerabilities with CodeQL

Request forwarde is a jump behavior in the server (note). It can ensure that the resources of the current application are accessed without changing the address bar of the browser, so as to share the data in a user request. Request forwarde can access WEB applications Resources in the root directory, such as resources in the `WEB-INF` directory, when some configuration files are stored in the `WEB-INF` directory, it is easy to cause sensitive information leakage.

![unsafe-url-forward](/images/unsafe-url-forward)

According to the `MVC` architecture design, the data requested by the user will be passed to the `Servlet` through the browser, and the program will process the business logic in the `Servlet`, and then return the processing result to the view. The process of returning the processing result to the view will often use request forwarde.

### ***Request forwarde vulnerability***

When the application does not verify the resource link requested to be forwarded, the resource leakage problem will occur. The following is the code that has the request forwarde vulnerability.

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

The application receives the user request data and performs logical processing, and then forwards the processed result to the internal resources of the current application as a response, and finally presents the result to the user. The above example shows the original request forwarde of `Servlet`, in Many `MVC` frameworks have already encapsulated request forwarde and request redirection.

### ***Prevent request forwarde vulnerabilities***

We know from the example above that what really causes the request forwarde vulnerability is actually giving the resource link to the user to control, so we want to prevent the occurrence of such vulnerabilities, and we only need to use the resource link as a fixed value.

### ***Use CodeQL to detect request forwarde vulnerabilities***

Before we use `CodeQL` to detect a vulnerability, we first need to clarify the Source, Sink, additional taint tracking configuration and disinfectant.

- source: direct remote user input.
- sink: (1) `javax.servlet.ServletRequest.getRequestDispatcher(sink);` (2)`javax.servlet.http.HttpServletRequest.getRequestDispatcher(sink);` (3)`jakarta.servlet.ServletRequest.getRequestDispatcher(sink );` (4)`jakarta.servlet.http.HttpServletRequest.getRequestDispatcher(sink);` (5)`Spring Mapping` method returns `String` and begins with `forward`, such as: "forward:"+url;(6 The Spring Mapping method returns an instance of org.springframework.web.servlet.ModelAndView.
- sanitizer: (1) The data stream is spliced, but does not start with `/WEB-INF/`; (2) The Spring Mapping method is spliced, but does not start with `forward`.

Write sink. I have annotated each class to facilitate understanding.

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

Write sanitizer

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

### ***Write at the back***

The `sink` and `sanitizer` that use `CodeQL` to detect request forwarde vulnerabilities also need to be continuously tuned to continuously reduce the rate of false negatives and false positives. In addition, you can use `CodeQL` to analyze some 0 days, and then explore some undiscovered vulnerabilities Yes.


### ***Reference***

[File Disclosure](https://vulncat.fortify.com/en/detail?id=desc.dataflow.java.file_disclosure_spring)