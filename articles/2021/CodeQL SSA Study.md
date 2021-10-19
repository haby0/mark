文件路径: codeql\java\ql\src\semmle\code\java\dataflow\SSA.qll


### 1 fieldAccessInCallable谓词

当两个约束条件同时满足时，成立：
（1）字段访问fa等于字段f；
（2）字段访问fa在函数或构造函数c中；
```qll
predicate fieldAccessInCallable(FieldAccess fa, Field f, Callable c) {
  f = fa.getField() and
  c = fa.getEnclosingCallable()
}
```

### 2 TSsaSourceVariable类型

TSsaSourceVariable类型可以被继承

TSsaSourceVariable类型符合四种情况：TLocalVar 或 TPlainField 或 TEnclosingField 或 TQualifiedField

```qll
cached
private newtype TSsaSourceVariable =
  TLocalVar(Callable c, LocalScopeVariable v) {
    c = v.getCallable() or c = v.getAnAccess().getEnclosingCallable()
  } or
  TPlainField(Callable c, Field f) {
    exists(FieldRead fr |
      fieldAccessInCallable(fr, f, c) and
      (fr.isOwnFieldAccess() or f.isStatic())
    )
  } or
  TEnclosingField(Callable c, Field f, RefType t) {
    exists(FieldRead fr | fieldAccessInCallable(fr, f, c) and fr.isEnclosingFieldAccess(t))
  } or
  TQualifiedField(Callable c, SsaSourceVariable q, InstanceField f) {
    exists(FieldRead fr | fieldAccessInCallable(fr, f, c) and fr.getQualifier() = q.getAnAccess())
  }
```

#### 2.1 TLocalVar（类似一个函数，但是没有返回值）
当两个约束条件满足一个时，成立：
（1）当局部作用域变量v在函数或构造函数c中；
（2）当局部作用域变量v使用处所在函数或构造函数是c时成立

#### 2.2 TPlainField（类似一个函数，但是没有返回值）
当两个约束条件满足一个时，成立：
（1）当读取字段访问fr被this或super调用，并且它访问的字段是f，并且在函数或构造函数c中；
（2）字段f是静态的，并且它是读取字段访问表达式访问的字段，并且在函数或构造函数c中；



