
# 数据流在Java中的使用

本文主要介绍本地数据流、全局数据流和污点跟踪.

## **本地数据流**


本地数据流是对方法内的数据流进行分析. 因为数据只在方法内流转,所以分析起来很快, 准确度也很高.


#### **使用本地数据流**

DataFlow 是本地数据流库模块, 该模块定义的Node类表示在数据流图中的节点. Nodes分为表达式节点(ExprNode)和参数节点(ParameterNode), 可以使用asExpr和asParameter在数据流节点和表达式/参数进行映射.

example:

```ql
import java
import semmle.code.java.dataflow.DataFlow

from DataFlow::Node node
select node.asParameter() //根据数据流节点获取参数
```

而```asParameter()```通过```this.(ExplicitParameterNode).getParameter()```获取参数. ```ExplicitParameterNode```是```ParameterNode```的子类.

或者可以使用 exprNode和parameterNode将表达式/参数转换为Node节点.

example:

```ql
import java
import semmle.code.java.dataflow.DataFlow

from Parameter p
select DataFlow::parameterNode(p)
```

```localFlowStep(Node nodeFrom, Node nodeTo)```保存了所有从```nodeFrom```节点到```nodeTo```节点的本地数据流. 可以使用+或者*, 或者```localFlow(Node node1, Node node2)```进行递归调用.

example:

```ql
import java
import semmle.code.java.dataflow.DataFlow

from DataFlow::Node source,DataFlow::Node sink
where
  DataFlow::localFlowStep(source, sink)
select source,sink
```

#### **使用本地污点跟踪**

本地污点跟踪通过包括非保留值的流程步骤来扩展本地数据流.例如:

```java
String temp = x;
String y = temp + ", " + temp;
```

如果x是被污染的字符串, 那y也被污染.

本地污染跟踪库位于TaintTracking模块中. 和本地数据流一样, ```localTaintStep（DataFlow :: Node nodeFrom，DataFlow :: Node nodeTo）```保存了所有从```nodeFrom```节点到```nodeTo```节点的污点流.可以使用+或者*, 或者```localTaint(DataFlow::Node src, DataFlow::Node sink)```进行递归调用.


example:

```ql
import java
import semmle.code.java.dataflow.DataFlow

from DataFlow::Node source,DataFlow::Node sink
where
  TaintTracking::localTaintStep(source, sink)
select source,sink
```


#### **例子**

查询```new FileReader(..)```文件名.

```java
import java

from Constructor fileReader, Call call
where
  fileReader.getDeclaringType().hasQualifiedName("java.io", "FileReader") and
  call.getCallee() = fileReader
select call.getArgument(0)
```

查找使用本地数据流查询所有流入该参数的表达式.

```ql
import java
import semmle.code.java.dataflow.DataFlow

from Constructor fileReader, Call call, Expr src
where
  fileReader.getDeclaringType().hasQualifiedName("java.io", "FileReader") and
  call.getCallee() = fileReader and
  DataFlow::localFlow(DataFlow::exprNode(src), DataFlow::exprNode(call.getArgument(0)))
select src
```

查找使用本地数据流查询所有流入该参数的方法参数.

```ql
import java
import semmle.code.java.dataflow.DataFlow

from Constructor fileReader, Call call, Parameter p
where
  fileReader.getDeclaringType().hasQualifiedName("java.io", "FileReader") and
  call.getCallee() = fileReader and
  DataFlow::localFlow(DataFlow::parameterNode(p), DataFlow::exprNode(call.getArgument(0)))
select p
```

查找对格式字符串未进行硬编码的格式函数的调用.

```ql
import java
import semmle.code.java.dataflow.DataFlow
import semmle.code.java.StringFormat

from StringFormatMethod format, MethodAccess call, Expr formatString
where
  call.getMethod() = format and
  call.getArgument(format.getFormatStringIndex()) = formatString and
  not exists(DataFlow::Node source, DataFlow::Node sink |
    DataFlow::localFlow(source, sink) and
    source.asExpr() instanceof StringLiteral and
    sink.asExpr() = formatString
  )
select call, "Argument to String format method isn't hard-coded."
```

## **全局数据流**

全局数据流跟踪整个程序的数据流. 但是准确率不如本地数据流, 并且需要花费较多的时间和内存.

#### **使用全局数据流**

可以通过扩展类来使用全局数据流库```DataFlow::Configuration```:

```ql
import semmle.code.java.dataflow.DataFlow

class MyDataFlowConfiguration extends DataFlow::Configuration {
  MyDataFlowConfiguration() { this = "MyDataFlowConfiguration" }

  override predicate isSource(DataFlow::Node source) {
    ...
  }

  override predicate isSink(DataFlow::Node sink) {
    ...
  }
}
```

谓词定义:

- isSource: 定义污染数据从何处流入
- isSink: 定位污染数据从何处流出
- isSanitizer: 可选, 限制污染数据
- isAdditionalTaintStep: 可选, 增加其他污染步骤

当我们进行全局数据流分析时, 如果程序执行中间过程某个函数对污染数据做了校验, 我们需要使用```isSanitizer```排除数据流边缘.如果source和sink不是同一个变量,我们使用```isAdditionalTaintStep```增加数据流边缘.

使用```hasFlow(DataFlow::Node source, DataFlow::Node sink)```进行污点跟踪分析.


#### **source**

CodeQL库中预定义了一些source. 例如:```RemoteFlowSource```定义了远程用户可控的数据源, 如果远程用户输入没有在```RemoteFlowSource```中定义, 可以通过继承```RemoteFlowSource```类来定义.


#### **例子**

远程用户可控的数据源做为```source```

```ql
import java
import semmle.code.java.dataflow.FlowSources

class MyTaintTrackingConfiguration extends TaintTracking::Configuration {
  MyTaintTrackingConfiguration() {
    this = "MyTaintTrackingConfiguration"
  }

  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource
  }

  override predicate isSink(DataFlow::Node sink) {
    none()
  }
}
```

#### **练习**

使用全局污点分析查询程序中使用```java.net.URL```的SSRF漏洞, 其中创建的URL实例只有一个参数.

bug example

```java
URL url = new URL(request.getParameter("url"));
url.openStream();
```

实现思路:
- 继承```ClassInstanceExpr```类. 确定实例为URl实例, 并且只接收一个参数.
- 继承```TaintTracking::Configuration```类, 重写```isSource```、```isSink```和```isAdditionalTaintStep```谓词
- 谓词```isSource```确定为```RemoteFlowSource```, 谓词```isSink```确定为URl实例并且调用了```openStream()```方法.
- 重写```isAdditionalTaintStep```谓词. source到第一个参数形成全局污点数据流, 第二个参数到sink形成全局污点数据流.

```ql
import java
import semmle.code.java.dataflow.FlowSources
import DataFlow::PathGraph

class UrlClassInstanceExpr extends ClassInstanceExpr {
  UrlClassInstanceExpr() {
    this.getConstructedType().hasQualifiedName("java.net", "URL") and
    this.getNumArgument() = 1
  }
}

class UrlSSRFConfig extends TaintTracking::Configuration {
  UrlSSRFConfig() { this = "UrlSSRFConfig" }

  override predicate isSource(DataFlow::Node source) { source instanceof RemoteFlowSource }

  override predicate isSink(DataFlow::Node sink) {
    exists(MethodAccess ma, Method m | ma.getMethod() = m |
      m.hasName("openStream") and
      m.getDeclaringType().hasQualifiedName("java.net", "URL") and
      ma.getQualifier() = sink.asExpr()
    )
  }

  override predicate isAdditionalTaintStep(DataFlow::Node pred, DataFlow::Node succ) {
    exists(UrlClassInstanceExpr ucie |
      ucie.getArgument(0) = pred.asExpr() and
      ucie = succ.asExpr()
    )
  }
}

from DataFlow::PathNode source, DataFlow::PathNode sink, UrlSSRFConfig conf
where conf.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "Url ssrf might include code from $@.", source.getNode(),
  "this user input"
```

