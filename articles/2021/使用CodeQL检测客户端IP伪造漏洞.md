# 使用CodeQL检测客户端IP伪造漏洞

## **描述**

在数据分析、风控、访问权限控制等等场景中,应用程序需要获取请求的客户端IP地址.但是,当获取客户端IP地址的方式不正确时,容易被攻击者利用,造成风控策略和访问控制被绕过等等危害.

下面我会使用真实的案例介绍不正确的获取方式以及如何修复.然后编写CodeQL在lgtm上检测github开源仓库.

### ***示例***

```java
PermissionFilter.java

@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
                throws ServletException, IOException {
        String uri = request.getRequestURI();
        if (StringUtils.containsIgnoreCase(uri, ".jsp")) {
                String ipAddr = RequestUtils.getIPAddress(request);
                if (!(StringUtils.equalsIgnoreCase(ipAddr, "localhost") || StringUtils.equalsIgnoreCase(ipAddr, "127.0.0.1")
                                || StringUtils.startsWith(ipAddr, "192.168."))) {
                        String contextPath = request.getContextPath();
                        String path = uri.substring(uri.indexOf(contextPath) + contextPath.length(), uri.length());
                        if (!includes.contains(path.toLowerCase())) {
                                logger.debug(RequestUtils.getIPAddress(request) + "试图访问->" + uri);
                                response.sendRedirect(request.getContextPath() + "/static/html/unauthorized.html");
                                return;
                        }
                }
        }
        filterChain.doFilter(request, response);
}


RequestUtils.java
public static String getIPAddress(HttpServletRequest request) {
    String ipAddress = request.getHeader("x-real-ip");
    if (StringUtils.isEmpty(ipAddress) || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getHeader("x-forwarded-for");
    }
    if (StringUtils.isEmpty(ipAddress) || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getHeader("Proxy-Client-IP");
    }
    if (StringUtils.isEmpty(ipAddress) || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getHeader("WL-Proxy-Client-IP");
    }
    if (StringUtils.isEmpty(ipAddress) || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getRemoteAddr();
    }
    if (ipAddress != null && ipAddress.length() > 15) {
            if (ipAddress.indexOf(",") > 0) {
                    ipAddress = ipAddress.substring(0, ipAddress.indexOf(","));
            }
    }
    return ipAddress;
}
```

上面是一个真实的[案例](https://github.com/gitee2008/jbpm/blob/a761cc1bf4f68b7b018054e85aa2c023f22f05d9/src/main/java/com/glaf/base/filter/PermissionFilter.java).通过程序逻辑可以知道,程序通过客户端IP地址进行访问权限控制.由于这里通过http标头(`X-Forwarded-For`或`X-Real-IP`或`Proxy-Client-IP`等等)获取客户端IP地址,攻击者可以伪造这些标识符的值来绕过访问权限控制.


### ***如何修复***

如果应用程序使用了反向代理,可以获取`X-Forwarded-For`标头值的最后一个值.

## **使用CodeQL检测客户端IP伪造漏洞**

明确source,sink和污染消毒剂.

- source: 从header指定标识符获取客户端IP地址
- sink: (1) 客户端IP地址是不是内网地址的if条件判断; (2) 客户端IP地址进行SQL操作
- sanitizer: 从header指定标识符获取客户端IP地址的最后一个值

(1) 编写source
```ql
import java
import DataFlow

class ClientSuppliedIpUsedInSecurityCheck extends DataFlow::Node {
  ClientSuppliedIpUsedInSecurityCheck() {
    exists(MethodAccess ma |
      ma.getMethod().hasName("getHeader") and
      ma.getArgument(0).(CompileTimeConstantExpr).getStringValue().toLowerCase() in [
          "x-forwarded-for", "x-real-ip", "proxy-client-ip", "wl-proxy-client-ip",
          "http_x_forwarded_for", "http_x_forwarded", "http_x_cluster_client_ip", "http_client_ip",
          "http_forwarded_for", "http_forwarded", "http_via", "remote_addr"
        ] and
      ma = this.asExpr()
    )
  }
}
```

(2) 编写sink
```ql
import java
import DataFlow
import semmle.code.java.frameworks.Networking
import semmle.code.java.security.QueryInjection

abstract class ClientSuppliedIpUsedInSecurityCheckSink extends DataFlow::Node { }

private class CompareSink extends ClientSuppliedIpUsedInSecurityCheckSink {
  CompareSink() {
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["equals", "equalsIgnoreCase"] and
      ma.getMethod().getDeclaringType() instanceof TypeString and
      ma.getMethod().getNumberOfParameters() = 1 and
      (
        ma.getArgument(0) = this.asExpr() and
        ma.getQualifier().(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
        not ma.getQualifier().(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
        or
        ma.getQualifier() = this.asExpr() and
        ma.getArgument(0).(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
        not ma.getArgument(0).(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
      )
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["contains", "startsWith"] and
      ma.getMethod().getDeclaringType() instanceof TypeString and
      ma.getMethod().getNumberOfParameters() = 1 and
      ma.getQualifier() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue().regexpMatch(getIpAddressRegex()) // Matches IP-address-like strings
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().hasName("startsWith") and
      ma.getMethod()
          .getDeclaringType()
          .hasQualifiedName(["org.apache.commons.lang3", "org.apache.commons.lang"], "StringUtils") and
      ma.getMethod().getNumberOfParameters() = 2 and
      ma.getAnArgument() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue().regexpMatch(getIpAddressRegex())
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["equals", "equalsIgnoreCase"] and
      ma.getMethod()
          .getDeclaringType()
          .hasQualifiedName(["org.apache.commons.lang3", "org.apache.commons.lang"], "StringUtils") and
      ma.getMethod().getNumberOfParameters() = 2 and
      ma.getAnArgument() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
      not ma.getAnArgument().(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
    )
  }
}

private class SqlOperationSink extends ClientSuppliedIpUsedInSecurityCheckSink {
  SqlOperationSink() { this instanceof QueryInjectionSink }
}

class SplitMethod extends Method {
  SplitMethod() {
    this.getNumberOfParameters() = 1 and
    this.hasQualifiedName("java.lang", "String", "split")
  }
}

string getIpAddressRegex() {
  result =
    "^((10\\.((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)(\\.)?)|(192\\.168\\.)|172\\.(1[6789]|2[0-9]|3[01])\\.)((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)(\\.)?((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)$"
}
```

(3) 编写sanitizer

```ql
import java
import DataFlow

class ClientSuppliedIpUsedInSecurityCheckSanitizer extends DataFlow::Node {
  ClientSuppliedIpUsedInSecurityCheckSanitizer() {
    exists(ArrayAccess aa, MethodAccess ma | aa.getArray() = ma |
      ma.getQualifier() = this.asExpr() and
      ma.getMethod() instanceof SplitMethod and
      not aa.getIndexExpr().(CompileTimeConstantExpr).getIntValue() = 0
    )
    or
    this.getType() instanceof PrimitiveType
    or
    this.getType() instanceof BoxedType
  }
}
```

## **Lgtm运行**

```ql
/**
 * @name IP address spoofing
 * @description A remote endpoint identifier is read from an HTTP header. Attackers can modify the value
 *              of the identifier to forge the client ip.
 * @kind path-problem
 * @problem.severity error
 * @precision high
 * @id java/ip-address-spoofing
 * @tags security
 *       external/cwe/cwe-348
 */

import java
import semmle.code.java.dataflow.FlowSources
import DataFlow::PathGraph

import DataFlow
import semmle.code.java.frameworks.Networking
import semmle.code.java.security.QueryInjection
import experimental.semmle.code.java.Logging

/**
 * A data flow source of the client ip obtained according to the remote endpoint identifier specified
 * (`X-Forwarded-For`, `X-Real-IP`, `Proxy-Client-IP`, etc.) in the header.
 *
 * For example: `ServletRequest.getHeader("X-Forwarded-For")`.
 */
class ClientSuppliedIpUsedInSecurityCheck extends DataFlow::Node {
  ClientSuppliedIpUsedInSecurityCheck() {
    exists(MethodAccess ma |
      ma.getMethod().hasName("getHeader") and
      ma.getArgument(0).(CompileTimeConstantExpr).getStringValue().toLowerCase() in [
          "x-forwarded-for", "x-real-ip", "proxy-client-ip", "wl-proxy-client-ip",
          "http_x_forwarded_for", "http_x_forwarded", "http_x_cluster_client_ip", "http_client_ip",
          "http_forwarded_for", "http_forwarded", "http_via", "remote_addr"
        ] and
      ma = this.asExpr()
    )
  }
}

/** A data flow sink for ip address forgery vulnerabilities. */
abstract class ClientSuppliedIpUsedInSecurityCheckSink extends DataFlow::Node { }

/**
 * A data flow sink for remote client ip comparison.
 *
 * For example: `if (!StringUtils.startsWith(ipAddr, "192.168.")){...` determine whether the client ip starts
 * with `192.168.`, and the program can be deceived by forging the ip address.
 */
private class CompareSink extends ClientSuppliedIpUsedInSecurityCheckSink {
  CompareSink() {
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["equals", "equalsIgnoreCase"] and
      ma.getMethod().getDeclaringType() instanceof TypeString and
      ma.getMethod().getNumberOfParameters() = 1 and
      (
        ma.getArgument(0) = this.asExpr() and
        ma.getQualifier().(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
        not ma.getQualifier().(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
        or
        ma.getQualifier() = this.asExpr() and
        ma.getArgument(0).(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
        not ma.getArgument(0).(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
      )
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["contains", "startsWith"] and
      ma.getMethod().getDeclaringType() instanceof TypeString and
      ma.getMethod().getNumberOfParameters() = 1 and
      ma.getQualifier() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue().regexpMatch(getIpAddressRegex()) // Matches IP-address-like strings
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().hasName("startsWith") and
      ma.getMethod()
          .getDeclaringType()
          .hasQualifiedName(["org.apache.commons.lang3", "org.apache.commons.lang"], "StringUtils") and
      ma.getMethod().getNumberOfParameters() = 2 and
      ma.getAnArgument() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue().regexpMatch(getIpAddressRegex())
    )
    or
    exists(MethodAccess ma |
      ma.getMethod().getName() in ["equals", "equalsIgnoreCase"] and
      ma.getMethod()
          .getDeclaringType()
          .hasQualifiedName(["org.apache.commons.lang3", "org.apache.commons.lang"], "StringUtils") and
      ma.getMethod().getNumberOfParameters() = 2 and
      ma.getAnArgument() = this.asExpr() and
      ma.getAnArgument().(CompileTimeConstantExpr).getStringValue() instanceof PrivateHostName and
      not ma.getAnArgument().(CompileTimeConstantExpr).getStringValue() = "0:0:0:0:0:0:0:1"
    )
  }
}

/** A data flow sink for sql operation. */
private class SqlOperationSink extends ClientSuppliedIpUsedInSecurityCheckSink {
  SqlOperationSink() { this instanceof QueryInjectionSink }
}

/** A method that split string. */
class SplitMethod extends Method {
  SplitMethod() {
    this.getNumberOfParameters() = 1 and
    this.hasQualifiedName("java.lang", "String", "split")
  }
}

string getIpAddressRegex() {
  result =
    "^((10\\.((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)(\\.)?)|(192\\.168\\.)|172\\.(1[6789]|2[0-9]|3[01])\\.)((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)(\\.)?((1\\d{2})?|(2[0-4]\\d)?|(25[0-5])?|([1-9]\\d|[0-9])?)$"
}

/**
 * Taint-tracking configuration tracing flow from obtaining a client ip from an HTTP header to a sensitive use.
 */
class ClientSuppliedIpUsedInSecurityCheckConfig extends TaintTracking::Configuration {
  ClientSuppliedIpUsedInSecurityCheckConfig() { this = "ClientSuppliedIpUsedInSecurityCheckConfig" }

  override predicate isSource(DataFlow::Node source) {
    source instanceof ClientSuppliedIpUsedInSecurityCheck
  }

  override predicate isSink(DataFlow::Node sink) {
    sink instanceof ClientSuppliedIpUsedInSecurityCheckSink
  }

  /**
   * Splitting a header value by `,` and taking an entry other than the first is sanitizing, because
   * later entries may originate from more-trustworthy intermediate proxies, not the original client.
   */
  override predicate isSanitizer(DataFlow::Node node) {
    exists(ArrayAccess aa, MethodAccess ma | aa.getArray() = ma |
      ma.getQualifier() = node.asExpr() and
      ma.getMethod() instanceof SplitMethod and
      not aa.getIndexExpr().(CompileTimeConstantExpr).getIntValue() = 0
    )
    or
    node.getType() instanceof PrimitiveType
    or
    node.getType() instanceof BoxedType
  }
}

from
  DataFlow::PathNode source, DataFlow::PathNode sink, ClientSuppliedIpUsedInSecurityCheckConfig conf
where conf.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "IP address spoofing might include code from $@.",
  source.getNode(), "this user input"
```

结果: https://lgtm.com/query/5856626224614437538/


## **参考**

[Prevent IP address spoofing with X-Forwarded-For header when using AWS ELB and Clojure Ring.](https://www.dennis-schneider.com/blog/prevent-ip-address-spoofing-with-x-forwarded-for-header-and-aws-elb-in-clojure-ring/)

[A Warning about X-Forwarded-For.](https://www.f5.com/company/blog/security-rule-zero-a-warning-about-x-forwarded-for)