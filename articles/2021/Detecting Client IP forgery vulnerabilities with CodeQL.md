# Detecting Client IP forgery vulnerabilities with CodeQL

## **Description**

In scenarios such as data analysis, risk control, and access control, the application needs to obtain the requested client IP address. However, when the way to obtain the client IP address is not correct, it is easy to be used by the attacker, resulting in the risk control strategy and access control bypassing and other hazards.

Below I will use a real case to introduce the incorrect acquisition method and how to fix it. Then write `CodeQL` to detect the github open source warehouse on `lgtm`.

### ***Example***

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

The above is a real [case](https://github.com/gitee2008/jbpm/blob/a761cc1bf4f68b7b018054e85aa2c023f22f05d9/src/main/java/com/glaf/base/filter/PermissionFilter.java). It can be known through program logic, The program uses the client's IP address for access control. Since here, the client's IP address is obtained through the http header (`X-Forwarded-For` or `X-Real-IP` or `Proxy-Client-IP`, etc.), Attackers can forge the values of these identifiers to bypass access control.


### ***How to fix**

If the application uses a reverse proxy, you can get the last value of the `X-Forwarded-For` header value.

## **Detecting Client IP forgery vulnerabilities with CodeQL**

Determine the source, sink and pollution disinfectant.

- source: Obtain the client IP address from the identifier specified in the header
- sink: (1) Whether the client IP address is the if condition of the intranet address is judged; (2) The client IP address performs SQL operations
- sanitizer: Obtain the last value of the client's IP address from the specified identifier in the header

(1) Write source
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

(2) Write sink
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

(3) Write sanitizer

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

## **Lgtm run**

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

result: https://lgtm.com/query/5856626224614437538/


## **Reference**

[Prevent IP address spoofing with X-Forwarded-For header when using AWS ELB and Clojure Ring.](https://www.dennis-schneider.com/blog/prevent-ip-address-spoofing-with-x-forwarded-for-header-and-aws-elb-in-clojure-ring/)

[A Warning about X-Forwarded-For.](https://www.f5.com/company/blog/security-rule-zero-a-warning-about-x-forwarded-for)