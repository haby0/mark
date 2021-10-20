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

#### 2.3 TEnclosingField（类似一个函数，但是没有返回值）
当两个约束条件同时满足时，成立：
（1）当读取字段访问fr访问的字段是f，并且在函数或构造函数c中；
（2）当读取字段访问fr访问的字段是类t的实例字段；

#### 2.4 TQualifiedField（类似一个函数，但是没有返回值）
当两个约束条件同时满足时，成立：
（1）当读取字段访问fr访问的字段是实例字段f，并且在函数或构造函数c中；
（2）当读取字段访问fr的限定符等于对`SsaSourceVariable`的访问；

### 3 SsaSourceVariable类

继承`TSsaSourceVariable`类型，符合`TSsaSourceVariable`中的四种情况

```qll
class SsaSourceVariable extends TSsaSourceVariable {
  // 获取与此`SsaSourceVariable`对应的变量。获取方法或构造函数中第一次使用到的变量。
  // 该变量对应三种类型：局部作用域变量、成员变量、静态变量、非静态实例变量
  Variable getVariable() {
    // this = TLocalVar(_, result) CodeQL语法，能够将result提取出来
    this = TLocalVar(_, result) or // result是局部作用域变量 
    this = TPlainField(_, result) or // result是成员变量或静态变量
    this = TEnclosingField(_, result, _) or
    this = TQualifiedField(_, _, result) // result是非静态实例变量
  }

  cached // 缓存
  // 获取方法或构造函数中使用变量的位置，不包含定义
  VarAccess getAnAccess() {
    // 返回使用局部作用域变量的变量
    exists(LocalScopeVariable v, Callable c |
      this = TLocalVar(c, v) and result = v.getAnAccess() and result.getEnclosingCallable() = c
    )
    or
    exists(Field f, Callable c | fieldAccessInCallable(result, f, c) |  // 限定字段变量在方法或构造函数中
      // 字段变量是成员变量或静态变量
      (result.(FieldAccess).isOwnFieldAccess() or f.isStatic()) and
      this = TPlainField(c, f)
      or
      // 字段变量是类实例变量
      exists(RefType t |
        this = TEnclosingField(c, f, t) and result.(FieldAccess).isEnclosingFieldAccess(t)
      )
      or
      // 字段变量是非静态实例变量
      exists(SsaSourceVariable q |
        result.getQualifier() = q.getAnAccess() and this = TQualifiedField(c, q, f)
      )
    )
  }
  
  // 获取定义了`SsaSourceVariable`的方法或构造函数。获取满足下面条件的字段变量所在的方法或构造函数
  // 1.局部作用域变量或局部作用域变量被使用所在方法或构造函数
  // 2.读取字段变量访问的成员变量或静态变量所在的方法或构造函数
  // 3.读取字段变量访问的变量所在的方法或构造函数
  // 4.读取字段变量访问的非静态实例变量所在的方法或构造函数
  Callable getEnclosingCallable() {
    this = TLocalVar(result, _) or
    this = TPlainField(result, _) or
    this = TEnclosingField(result, _, _) or
    this = TQualifiedField(result, _, _)
  }

  // 获取`SsaSourceVariable`的表示文本。获取字段变量的全限定名
  string toString() {
    exists(LocalScopeVariable v, Callable c | this = TLocalVar(c, v) |
      if c = v.getCallable()
      then result = v.getName()
      else result = c.getName() + "(..)." + v.getName()
    )
    or
    result = this.(SsaSourceField).ppQualifier() + "." + getVariable().toString()
  }

  // 获取变量在方法或构造函数中第一次被使用的位置
  private VarAccess getFirstAccess() {
    result =
      min(this.getAnAccess() as a
        order by
          a.getLocation().getStartLine(), a.getLocation().getStartColumn()
      )
  }

  // 获取变量位置
  Location getLocation() {
    exists(LocalScopeVariable v | this = TLocalVar(_, v) and result = v.getLocation())
    or
    this instanceof SsaSourceField and result = getFirstAccess().getLocation()
  }

  // 获取变量类型
  Type getType() { result = this.getVariable().getType() }

  // 获取变量限定符
  SsaSourceVariable getQualifier() { this = TQualifiedField(_, result, _) }

  // 获取SSA变量，该变量将此变量做为其基础源变量
  SsaVariable getAnSsaVariable() { result.getSourceVariable() = this }
}
```
