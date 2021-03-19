# Detecting XQuery injection with CodeQL

## **Preface**

In this article, I will introduce how to use `CodeQL` to detect `XQuery` injection.

First, I will introduce `XQuery` injection. Second, I will use examples to illustrate the hazards of `Xquery` injection and how to prevent it. Finally, I will introduce how to write `ql` to detect `XQuery` injection.

## **XQuery injection**

`XQuery` is a functional language that can retrieve information stored in `XML` format, including elements and attributes in `XML` documents, and is often used to extract information from web services. Similar to `XPath` injection, `XQuery` injection attacks take advantage of the looseness of the `XQuery` parser Input and fault tolerance features can load malicious code in URLs and forms to enter programs. When information containing malicious code participates in the dynamic construction of `XQuery` query expressions, it will cause `XQuery `injection. Attackers can use vulnerabilities to obtain sensitive information and gain system permissions. For a detailed description of the defect, please refer to [CWE-652: Improper Neutralization of Data within XQuery Expressions ('XQuery Injection')](http://cwe.mitre.org/data/definitions/652.html).

## **The harm of XQuery injection**

### ***Example***

```java
String name = request.getParameter("name");

XQDataSource ds = new SaxonXQDataSource();

XQConnection conn = ds.getConnection();

String query = "for $user in doc(\"users.xml\")/Users/User[name='" + name

        + "'] return $user/password";

XQPreparedExpression xqpe = conn.prepareExpression(query);

XQResultSequence result = xqpe.executeQuery();

while (result.next()) {

    System.out.println(result.getItemAsString(null));

}
```

The above code is to obtain the user name from the request and then splice it into the expression to be queried, and finally output the user password. Since the user name can be controlled by the user, when the user enters the user name `admin' or''='`, we the query expression is:

```sql
for $user in doc(\"users.xml\")/Users/User[name='admin' or ''=''] return $user/password
```

According to the query expression, we know that this query will always be `true` and will eventually return the passwords of all users, which causes `XQuery` injection.

### ***Fix code***

```java
XQDataSource xqds = new SaxonXQDataSource();

XQConnection conn;

try {

    String name = request.getParameter("name");

    String query = "declare variable $name as xs:string external;"

            + " for $user in doc(\"users.xml\")/Users/User[name=$name] return $user/password";

    conn = xqds.getConnection();

    XQExpression expr = conn.createExpression();

    expr.bindString(new QName("name"), name,

            conn.createAtomicType(XQItemType.XQBASETYPE_STRING));

    XQResultSequence result = expr.executeQuery(query);

    while (result.next()) {

        System.out.println(result.getItemAsString(null));

    }

} catch (XQException e) {

    e.printStackTrace();

}
```

In the above repair code, declare the string variable `name` by using the `declare` syntax. Then use the `bindString()` method of the `XQExpression` object to bind the variable to the query. Similar to the placeholder in the `SQL` query , Here use declaration variables to simulate parameterized queries.

## **Detecting XQuery injection with CodeQL**

[CodeQL](https://securitylab.github.com/tools/codeql) is an industry-leading semantic code analysis engine, we can use it to find vulnerabilities in the entire code base. In addition, it is also a free research and open source software.

We can define the source and sink, and then let CodeQL help us with global taint tracking and data flow analysis.In `XQuery` injection:

- The source is user remote input, and the `semmle.code.java.dataflow.FlowSources` library in `CodeQL` has helped us define the remote input source `RemoteFlowSource`, which is very convenient.
- The receiver is the `XQPreparedExpression` object in `XQPreparedExpression.executeQuery()`.
- But this is not enough, `RemoteFlowSource` and `XQPreparedExpression` are not the same data, we need to define the `isAdditionalTaintStep` predicate to tell the `CodeQL` engine other pollution propagation steps.

According to our ideas, a `ql` was written.

```ql
/**

 * @name XQuery query built from user-controlled sources

 * @description Building an XQuery query from user-controlled sources is vulnerable to insertion of

 *              malicious XQuery code by the user.

 * @kind path-problem

 * @problem.severity error

 * @precision high

 * @id java/xquery-injection

 * @tags security

 *       external/cwe/cwe-652

 */

import java

import semmle.code.java.dataflow.FlowSources

import XQueryInjectionLib

import DataFlow::PathGraph



class XQueryPreparedExecuteCall extends MethodAccess {

  XQueryPreparedExecuteCall() {

    exists(Method m |

      this.getMethod() = m and

      m.hasName("executeQuery") and

      m.getDeclaringType()

          .getASourceSupertype*()

          .hasQualifiedName("javax.xml.xquery", "XQPreparedExpression")

    )

  }



  /** Return this prepared expression. */

  Expr getPreparedExpression() { result = this.getQualifier() }

}



class XQueryInjectionConfig extends TaintTracking::Configuration {

  XQueryInjectionConfig() { this = "XQueryInjectionConfig" }



  override predicate isSource(DataFlow::Node source) { source instanceof RemoteFlowSource }



  override predicate isSink(DataFlow::Node sink) {

    sink.asExpr() = any(XQueryPreparedExecuteCall xpec).getPreparedExpression()

  }



  override predicate isAdditionalTaintStep(DataFlow::Node pred, DataFlow::Node succ) {

    exists(XQueryParserCall parser | pred.asExpr() = parser.getInput() and succ.asExpr() = parser)

  }

}



from DataFlow::PathNode source, DataFlow::PathNode sink, XQueryInjectionConfig conf

where conf.hasFlowPath(source, sink)

select sink.getNode(), source, sink, "XQuery query might include code from $@.", source.getNode(),

  "this user input"
```

This `ql` mainly contains several parts:

- `isSource()` defines the source, usually remote user input, just use the standard `RemoteFlowSource` class directly.
- `isSink()` defines the sink.
- `isAdditionalTaintStep()` defines an additional taint propagation step.

In order for our query results to show the entire taint tracking and data flow, we need to add `@kind path-problem` to `ql`. Then our query is written. But this is not enough, because we need to correct `XQuery Inject `to study carefully and find other code calls that may have security issues.

## **Reference**

[Balisage](https://www.balisage.net/Proceedings/vol7/html/Vlist02/BalisageVol7-Vlist02.html)

[Writing CodeQL queries](https://codeql.github.com/docs/writing-codeql-queries/)