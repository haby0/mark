### 前言

这些`CSV`（实际上是`SSV` - 分号分隔）行中的最后一个条目是污点或值，具体取决于模型是否传播污点或参数或限定符的确切值。

CodeQL中的CSV flow模型（模型即数据）。CodeQL可以基于AST对某种风险建模，也可以使用CSV方式建模，在某些场景下CSV方式建模更方便、更流畅。


### 示例说明

用SSRF的例子

```ql
private class UrlOpenSinkAsRequestForgerySink extends RequestForgerySink {
  UrlOpenSinkAsRequestForgerySink() { sinkNode(this, "open-url") }
}
```

```ql
InterpretNode类型：包含了所有节点。

cached
predicate sinkNode(Node node, string kind) {
  exists(InterpretNode n | isSinkNode(n, kind) and n.asNode() = node)
}
```

```ql
private class ApacheHttpOpenUrlSink extends SinkModelCsv {
  override predicate row(string row) {
    row =
      [
        "org.apache.http;HttpRequest;true;setURI;;;Argument[0];open-url",
        "org.apache.http.message;BasicHttpRequest;false;BasicHttpRequest;(RequestLine);;Argument[0];open-url",
        "org.apache.http.message;BasicHttpRequest;false;BasicHttpRequest;(String,String);;Argument[1];open-url",
        "org.apache.http.message;BasicHttpRequest;false;BasicHttpRequest;(String,String,ProtocolVersion);;Argument[1];open-url",
        "org.apache.http.message;BasicHttpEntityEnclosingRequest;false;BasicHttpEntityEnclosingRequest;(RequestLine);;Argument[0];open-url",
        "org.apache.http.message;BasicHttpEntityEnclosingRequest;false;BasicHttpEntityEnclosingRequest;(String,String);;Argument[1];open-url",
        "org.apache.http.message;BasicHttpEntityEnclosingRequest;false;BasicHttpEntityEnclosingRequest;(String,String,ProtocolVersion);;Argument[1];open-url",
        "org.apache.http.client.methods;HttpGet;false;HttpGet;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpHead;false;HttpHead;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpPut;false;HttpPut;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpPost;false;HttpPost;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpDelete;false;HttpDelete;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpOptions;false;HttpOptions;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpTrace;false;HttpTrace;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpPatch;false;HttpPatch;;;Argument[0];open-url",
        "org.apache.http.client.methods;HttpRequestBase;true;setURI;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;setUri;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;get;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;post;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;put;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;options;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;head;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;delete;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;trace;;;Argument[0];open-url",
        "org.apache.http.client.methods;RequestBuilder;false;patch;;;Argument[0];open-url"
      ]
  }
}

private predicate sinkModelCsv(string row) {
  row =
    [
      // Open URL
      "java.net;URL;false;openConnection;;;Argument[-1];open-url",
      "java.net;URL;false;openStream;;;Argument[-1];open-url",
      "java.net.http;HttpRequest;false;newBuilder;;;Argument[0];open-url",
      "java.net.http;HttpRequest$Builder;false;uri;;;Argument[0];open-url",
      "java.net;URLClassLoader;false;URLClassLoader;(URL[]);;Argument[0];open-url",
      "java.net;URLClassLoader;false;URLClassLoader;(URL[],ClassLoader);;Argument[0];open-url",
      "java.net;URLClassLoader;false;URLClassLoader;(URL[],ClassLoader,URLStreamHandlerFactory);;Argument[0];open-url",
      "java.net;URLClassLoader;false;URLClassLoader;(String,URL[],ClassLoader);;Argument[1];open-url",
      "java.net;URLClassLoader;false;URLClassLoader;(String,URL[],ClassLoader,URLStreamHandlerFactory);;Argument[1];open-url",
      "java.net;URLClassLoader;false;newInstance;;;Argument[0];open-url",
      // Create file
      "java.io;FileOutputStream;false;FileOutputStream;;;Argument[0];create-file",
      "java.io;RandomAccessFile;false;RandomAccessFile;;;Argument[0];create-file",
      "java.io;FileWriter;false;FileWriter;;;Argument[0];create-file",
      "java.nio.file;Files;false;move;;;Argument[1];create-file",
      "java.nio.file;Files;false;copy;;;Argument[1];create-file",
      "java.nio.file;Files;false;newOutputStream;;;Argument[0];create-file",
      "java.nio.file;Files;false;newBufferedReader;;;Argument[0];create-file",
      "java.nio.file;Files;false;createDirectory;;;Argument[0];create-file",
      "java.nio.file;Files;false;createFile;;;Argument[0];create-file",
      "java.nio.file;Files;false;createLink;;;Argument[0];create-file",
      "java.nio.file;Files;false;createSymbolicLink;;;Argument[0];create-file",
      "java.nio.file;Files;false;createTempDirectory;;;Argument[0];create-file",
      "java.nio.file;Files;false;createTempFile;;;Argument[0];create-file",
      // Bean validation
      "javax.validation;ConstraintValidatorContext;true;buildConstraintViolationWithTemplate;;;Argument[0];bean-validation",
      // Set hostname
      "javax.net.ssl;HttpsURLConnection;true;setDefaultHostnameVerifier;;;Argument[0];set-hostname-verifier",
      "javax.net.ssl;HttpsURLConnection;true;setHostnameVerifier;;;Argument[0];set-hostname-verifier"
    ]
}

class SinkModelCsv extends Unit {
  /** Holds if `row` specifies a sink definition. */
  abstract predicate row(string row);
}

// sink模型。有两种方式：（1）在 sinkModelCsv 谓词中直接描述sink；（2）继承 SinkModelCsv 类，重写 row 谓词。
private predicate sinkModel(string row) {
  sinkModelCsv(row) or
  any(SinkModelCsv s).row(row)
}

// sink模型规范：`namespace; type; subtypes; name; signature; ext; input; kind`
// 1、namespace：包名
// 2、type：包中的类名。其中内部类格式：A$B
// 3、subtypes：布尔类型。[true, false]。当为true时，会寻找子类重写的类型成员name。
// 4、name：可选，类型成员。成员可以是构造函数、方法、字段和嵌套类型
// 5、signature：可选，限制类中的成员。格式为用括号括起来的类型的逗号分隔列表。类型可以是字面量或者完全限定名。signature也可以为空。()和""得到的结果一致。完全限定名示例：void clear(Transaction tx)，signature可以写成(org.apache.activemq.store.kahadb.disk.page.Transaction)
// 6、ext：额外的边，只有""和"Annotated"两个值
// 7、input：指定数据如何进入前6列选择的元素。可以是：""、"Argument[n]"、"Argument[n1..n2]"、"ReturnValue"
// 8、kind：标记上面的最终标识的elements用于哪类建模。例如："open-url"用于ssrf漏洞sink建模
predicate sinkModel(
  string namespace, string type, boolean subtypes, string name, string signature, string ext,
  string input, string kind
) {
  exists(string row |
    sinkModel(row) and
    row.splitAt(";", 0) = namespace and
    row.splitAt(";", 1) = type and
    row.splitAt(";", 2) = subtypes.toString() and
    subtypes = [true, false] and
    row.splitAt(";", 3) = name and
    row.splitAt(";", 4) = signature and
    row.splitAt(";", 5) = ext and
    row.splitAt(";", 6) = input and
    row.splitAt(";", 7) = kind
  )
}
```

```ql

// 获取可调用对象形参类型擦除字符串表示。 例如：public HashSet(Collection<? extends E> c) { .. } 依次返回"("，"Collection"，")"
// 关于Java类型擦除，可以查看：https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.6
private string paramsStringPart(Callable c, int i) {
  i = -1 and result = "("
  or
  exists(int n, string p | c.getParameterType(n).getErasure().toString() = p |
    i = 2 * n and result = p
    or
    i = 2 * n - 1 and result = "," and n != 0
  )
  or
  i = 2 * c.getNumberOfParameters() and result = ")"
}

// 获取可调用对象形参类型擦除列表。例如：public HashSet(Collection<? extends E> c) { .. } 得到 (Collection)
cached
string paramsString(Callable c) { result = concat(int i | | paramsStringPart(c, i) order by i) }

// 收敛计算结果
pragma[nomagic]
private predicate elementSpec(
  string namespace, string type, boolean subtypes, string name, string signature, string ext
) {
  sourceModel(namespace, type, subtypes, name, signature, ext, _, _) or
  sinkModel(namespace, type, subtypes, name, signature, ext, _, _) or
  summaryModel(namespace, type, subtypes, name, signature, ext, _, _, _)
}

// 1、interpretElement0方法首先调用elementSpec收敛计算结果
// 2、通过exists得到Element节点
// 2.1、判断存在指定的引用类型t。如果引用类型t都不存在，就没必要做下面的判断了。自顶而下。
// 2.2、两步计算获取到Element节点
// 2.2.1、引用类型t中存在类型成员m。当subtypes为true时，result可以是类型成员m的重写。并且需要满足下面条件之一：
//       （1）signature等于"";
//       （2）类型成员m签名以signature结尾；
//       （3）signature等于可调用对象形参类型擦除列表；
// 2.2.2、当引用类型t中不存在类型成员m时。首先，name等于""，signature等于""。当subtypes为true时，Element节点超类等于引用类型t。当subtypes为false时，Element节点等于引用类型t

// 所以interpretElement0方法会得到两种结果。一种是类型成员或类型成员的重写，一种是引用类型或子类型。
private Element interpretElement0(
  string namespace, string type, boolean subtypes, string name, string signature
) {
  elementSpec(namespace, type, subtypes, name, signature, _) and // 这里粗看是多余的，和CodeQL研发交流，这里是为了收敛谓词计算结果。在MyBatis SQL注入建模时遇到笛卡尔积的问题，和这里类似。
  exists(RefType t | t.hasQualifiedName(namespace, type) |  // 引用类型t在namespace包type名称中。
    exists(Member m |  // 当subtypes为true时，会得到所有的子类重写方法，包括自身方法。当subtypes为false时，只能得到自身方法
      (
        result = m
        or
        subtypes = true and result.(SrcMethod).overridesOrInstantiates+(m)
      ) and
      m.getDeclaringType() = t and  // 声明类型成员的类型等于引用类型t
      m.hasName(name)  // 类型成员名为name
    |
      signature = "" or  // signature可以为""，在()内使用逗号分隔的完全限定名列表，字面量
      m.(Callable).getSignature() = any(string nameprefix) + signature or
      paramsString(m) = signature // signature等于可调用对象形参类型擦除列表
    )
    or
    (if subtypes = true then result.(SrcRefType).getASourceSupertype*() = t else result = t) and  // 当subtypes为true时，返回类型的自身和所有超类等于引用类型t。否则返回类型等于引用类型t
    name = "" and  // name等于""
    signature = ""  // signature等于""
  )
}

/** Gets the source/sink/summary element corresponding to the supplied parameters. */
// 获取与提供的参数对应的 source/sink/summary 元素

// 当ext等于""时，得到类型成员，类型成员的复写，引用类型，引用类型子类型
// 当ext等于"Annotated"时，得到使用注解引用类型或引用类型子类型的引用类型
Element interpretElement(
  string namespace, string type, boolean subtypes, string name, string signature, string ext
) {
  elementSpec(namespace, type, subtypes, name, signature, ext) and // 这里粗看是多余的，和CodeQL研发交流，这里是为了收敛谓词计算结果。在MyBatis SQL注入建模时遇到笛卡尔积的问题，和这里类似。
  exists(Element e | e = interpretElement0(namespace, type, subtypes, name, signature) |  // interpretElement0得到类型成员，类型成员的复写，引用类型，引用类型子类型
    ext = "" and result = e
    or
    ext = "Annotated" and result.(Annotatable).getAnAnnotation().getType() = e
  )
}

predicate sinkElement(SourceOrSinkElement e, string input, string kind) {
  exists(
    string namespace, string type, boolean subtypes, string name, string signature, string ext
  |
    sinkModel(namespace, type, subtypes, name, signature, ext, input, kind) and  // 通过kind标签，在sinkModel谓词得到namespace, type, subtypes, name, signature, ext, input
    e = interpretElement(namespace, type, subtypes, name, signature, ext) // 通过上步获取到的namespace, type, subtypes, name, signature, ext得到一个解释的element，element得到类型成员，类型成员的复写，引用类型，引用类型子类型，使用注解引用类型或引用类型子类型的引用类型
  )
}
```

```ql
string specLast(string s) {
  exists(int len |
    specLength(s, len) and
    specSplit(s, result, len - 1)
  )
}

private predicate sinkElementRef(InterpretNode ref, string input, string kind) {
  exists(SourceOrSinkElement e |
    sinkElement(e, input, kind) and // sinkElement方法根据kind得到SourceOrSinkElement节点和input值。可能是：类型成员，类型成员的复写，引用类型，引用类型子类型，使用注解引用类型或引用类型子类型的引用类型
    if inputNeedsReference(specLast(input))
    then e = ref.getCallTarget()
    else e = ref.asElement()
  )
}

predicate isSinkNode(InterpretNode node, string kind) {
  exists(InterpretNode ref, string input |
    sinkElementRef(ref, input, kind) and
    interpretInput(input, 0, ref, node)
  )
}
```

```ql
Top包含了所有节点，例如：Method，class，file等。

class SourceOrSinkElement = Top;

private newtype TInterpretNode =
  TElement(SourceOrSinkElement n) or
  TNode(Node n)
  
class InterpretNode extends TInterpretNode {
  /** Gets the element that this node corresponds to, if any. */
  SourceOrSinkElement asElement() { this = TElement(result) }

  /** Gets the data-flow node that this node corresponds to, if any. */
  Node asNode() { this = TNode(result) }

  /** Gets the call that this node corresponds to, if any. */
  DataFlowCall asCall() { result.asCall() = this.asElement() }

  /** Gets the callable that this node corresponds to, if any. */
  DataFlowCallable asCallable() { result.asCallable() = this.asElement() }

  /** Gets the target of this call, if any. */
  Callable getCallTarget() { result = this.asCall().asCall().getCallee().getSourceDeclaration() }

  /** Gets a textual representation of this node. */
  string toString() {
    result = this.asElement().toString()
    or
    result = this.asNode().toString()
  }

  /** Gets the location of this node. */
  Location getLocation() {
    result = this.asElement().getLocation()
    or
    result = this.asNode().getLocation()
  }
}
```
