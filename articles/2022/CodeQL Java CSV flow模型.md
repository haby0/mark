### 1. 前言

在CodeQL for Java中, 因为我们对抽象语法树很熟悉, 所以在对某种风险建模时经常基于`AST`写`source`, `sink`, `taint`. CodeQL for Java中还有一种CSV flow model, 让我们可以通过CSV row的方式写`source`, `sink`, `taint`. 相比较`AST`建模方式, 大多数情况下CSV row更加方便灵活.

CSV row示例:

```ql
// Open URL
"java.net;URL;false;openConnection;;;Argument[-1];open-url"
"java.net;URL;false;openStream;;;Argument[-1];open-url"
```

### 2. CSV flow model

名词解释：
CSV: 实际上是SSV(semicolon separated values), 中文意思是分号分隔值.

使用CodeQL去分析一些程序时, 实际上底层已经用到了CSV row, 下面做一个测试.

写一段Java程序代码并创建CodeQL数据库.

```java
import javax.servlet.http.HttpServletRequest;

@RequestMapping("/file1")
public void testDeletebad1(HttpServletRequest request) throws Exception {
	String rName = request.getParameter("reportName");
	File rFile = new File(rName);
	rFile.delete();

}
```

对任意文件删除建模

```ql
/**
 * @name Uncontrolled data used in path expression
 * @description Accessing paths influenced by users can allow an attacker to access unexpected resources.
 * @kind path-problem
 * @problem.severity error
 * @security-severity 7.5
 * @precision high
 * @id java/path-injection
 * @tags security
 *       external/cwe/cwe-022
 *       external/cwe/cwe-023
 *       external/cwe/cwe-036
 *       external/cwe/cwe-073
 */

import java
import semmle.code.java.dataflow.FlowSources
import DataFlow::PathGraph

class PathInjectionConfig extends TaintTracking::Configuration {
  PathInjectionConfig() { this = "PathInjectionConfig" }

  override predicate isSource(DataFlow::Node source) { source instanceof RemoteFlowSource }

  override predicate isSink(DataFlow::Node sink) {
    exists(MethodAccess ma |
      ma.getMethod().hasName("delete") and
      ma.getQualifier() = sink.asExpr()
    )
  }
}

from DataFlow::PathNode source, DataFlow::PathNode sink, PathInjectionConfig conf
where conf.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "$@ flows to here and is used in a path.", source.getNode(),
  "user-provided value"
```

运行的结果是:

![1](/articles/2022/images/CodeQL-Java-CSV-flow模型/1.jpg)


接着注释掉ql/lib/semmle/code/java/dataflow/ExternalFlow.qll中下面的两行内容.

```ql
"java.io;File;false;File;;;Argument[0];Argument[-1];taint",
"java.io;File;false;File;;;Argument[1];Argument[-1];taint",
```


再次运行, 运行结果是:
![2](/articles/2022/images/CodeQL-Java-CSV-flow模型/2.jpg)


出现这种情况的原因是: CodeQL在`summaryModelCsv`谓词和继承`SummaryModelCsv`类重写`row`谓词中增加一些附加的污点传播, 并通过CSV row最后一个分号后的`taint`字符标识. 

远程污点分析经常写的一条语句 `source instanceof RemoteFlowSource`, 在`sourceModelCsv`谓词和继承`SourceModelCsv`类重写`row`谓词中做了定义, 并通过CSV row最后一个分号后的`remote`字符标识.

#### 2.1. CSV规范

在ql/lib/semmle/code/java/dataflow/ExternalFlow.qll#L6-L12中可以找到CSV规范

```ql
The CSV specification has the following columns:
- Sources:
  `namespace; type; subtypes; name; signature; ext; output; kind`
- Sinks:
  `namespace; type; subtypes; name; signature; ext; input; kind`
- Summaries:
  `namespace; type; subtypes; name; signature; ext; input; output; kind`
```

名词解释:
* namespace: 包名
* type: 包中的类名, 可以是内部类. 内部类使用的格式`A$B`
* subtypes：布尔类型,可以取值为true或false. 当为true时, 会寻找子类重写的类型成员name(使用`overridesOrInstantiates`谓词得到). 否则寻找当前类类型成员name
* name: 类型成员, 可选项. 类型成员可以是构造函数、方法
* signature: 类型成员签名, 用来限定类型成员, 可选项 signature的格式是用括号括起来的类型(可以是字面量或者完全限定名)的逗号分隔列表. 示例：`void clear(File file)`, signature字面量表示:`(File)`, signature完全限定名表示:`(java.io.File)`
* ext: 额外的边, 现在只有`""`和`Annotated`两个值. 一般场景下使用不到. 当使用`Annotated`时, `namespace.type`是一个注解. 例如:[PlainSQL注解](https://github.com/jOOQ/jOOQ/blob/main/jOOQ/src/main/java/org/jooq/PlainSQL.java#L64)
* input：指定污点如何进入前6列选择的元素, 可以是: `""`, `Argument[n]`, `Argument[n1..n2]`, `ReturnValue`. 在定义sink和额外污点传播时使用.
* output: 可以是`""`, `Argument[n]`,`Argument[n1..n2]`, `Parameter`, `Parameter[n]`, `Parameter[n1..n2]`, `ReturnValue`
* kind: 标记上面的最终标识的类型成员用于哪类建模. 例如: `open-url`用于`ssrf`漏洞建模

注: 
1. 目前还无法直接根据一个`Callable`直接得到CSV row, `CodeQL`并没有提供这个工具类.
2. 当kind等于`remote`或`taint`时, 是一个全局定义, 在对程序污点分析时, 会全局生效.

#### 2.2. CSV row的使用

用SSRF规则说明, `RequestForgerySink`抽象类一个子类是这么定义的:
```ql
文件位置:ql/lib/semmle/code/java/security/RequestForgery.qll#L40-L42

private class UrlOpenSinkAsRequestForgerySink extends RequestForgerySink {
  UrlOpenSinkAsRequestForgerySink() { sinkNode(this, "open-url") }
}
```

`sinkNode`谓词的定义如下:

```ql
文件位置:ql/lib/semmle/code/java/dataflow/ExternalFlow.qll#L723-L726

cached
predicate sinkNode(Node node, string kind) {
    exists(InterpretNode n | isSinkNode(n, kind) and n.asNode() = node)
}
```

`isSinkNode`谓词通过传入`InterpretNode`节点和`"open-url"`, CodeQL程序会找到所有kind为`"open-url"`的CSV row(在`sinkModelCsv`谓词和继承`SinkModelCsv`类重写`row`谓词中), 然后从程序中找到所有符合CSV flow模型定义的`InterpretNode`节点.

所以我们在使用CSV row建模sink时, 可以直接使用`sinkNode`谓词, 传入CSV row中的kind即可.


### 3. 结语

CSV flow模型一定程度上让建模更加方便, 比如针对某一个框架建模, 不需要再去写AST.

参考:
https://github.com/github/codeql/blob/main/java/ql/lib/semmle/code/java/dataflow/ExternalFlow.qll

https://github.com/github/codeql/blob/main/java/ql/lib/semmle/code/java/dataflow/internal/FlowSummaryImpl.qll

https://github.com/github/codeql/blob/main/java/ql/lib/semmle/code/java/dataflow/internal/FlowSummaryImplSpecific.qll
