### 前言

这些`CSV`（实际上是`SSV` - 分号分隔）行中的最后一个条目是污点或值，具体取决于模型是否传播污点或参数或限定符的确切值

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

// sink模型
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
predicate sinkElement(SourceOrSinkElement e, string input, string kind) {
  exists(
    string namespace, string type, boolean subtypes, string name, string signature, string ext
  |
    sinkModel(namespace, type, subtypes, name, signature, ext, input, kind) and  // 根据sinkModel谓词得到
    e = interpretElement(namespace, type, subtypes, name, signature, ext)
  )
}
```

```ql
private predicate sinkElementRef(InterpretNode ref, string input, string kind) {
  exists(SourceOrSinkElement e |
    sinkElement(e, input, kind) and
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
