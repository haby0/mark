codeql\java\ql\src\semmle\code\java\Variable.qll

对Java变量建模，包括：局部变量、成员变量、类变量

形参：局部变量


### 1 Variable类
```ql
// 变量:字段、局部变量或参数
class Variable extends @variable, Annotatable, Element, Modifiable {
  // 变量类型，
  /*abstract*/ Type getType() { none() }

  // 获取对此变量的访问权限。例如：private String test;得到 test
  VarAccess getAnAccess() { variableBinding(result, this) }

  // 获取变量赋值右侧的表达式。
  Expr getAnAssignedValue() {
    exists(LocalVariableDeclExpr e | e.getVariable() = this and result = e.getInit()) // LocalVariableDeclExpr e.getVariable():得到局部变量声明。LocalVariableDeclExpr e.getInit():获取此局部变量声明表达式的初始化表达式。例如:String code = request.getParameter("code"); e->code，result->request.getParameter("code")，this->String code = request.getParameter("code");
    or
    exists(AssignExpr e | e.getDest() = this.getAnAccess() and result = e.getSource()) // AssignExpr e.getDest():简单赋值表达式左侧。AssignExpr e.getSource():简单赋值表达式右侧。例如：remoteAddr = httpServletRequest.getHeader("X-FORWARDED-FOR"); 得到httpServletRequest.getHeader("X-FORWARDED-FOR")
  }

  // 获取此变量的初始化表达式。
  Expr getInitializer() { none() }

  // 显示此变量的string字符串。
  string pp() { result = this.getType().getName() + " " + this.getName() }
}
```

### 2 LocalScopeVariable类
```ql
class LocalScopeVariable extends Variable, @localscopevariable {
  // 获取声明此变量的可调用对象
  abstract Callable getCallable();
}
```

### 3 LocalVariableDecl类
```ql
// 局部变量声明
class LocalVariableDecl extends @localvar, LocalScopeVariable {
  // 获取局部变量类型
  override Type getType() { localvars(this, _, result, _) }

  // 获取声明此变量的局部表达式。除了变量类型外的局部表达式。例如：String code = request.getParameter("code");得到code = request.getParameter("code");
  LocalVariableDeclExpr getDeclExpr() { localvars(this, _, _, result) }

  // 获取局部变量的父表达式。例如：String code = request.getParameter("code");得到code = request.getParameter("code");
  Expr getParent() { localvars(this, _, _, result) }

  // 获取发生此声明的可调用对象，一般是当前方法
  override Callable getCallable() { result = this.getParent().getEnclosingCallable() }

  // 获取发生此声明的可调用对象，一般是当前方法
  Callable getEnclosingCallable() { result = getCallable() }

  // 显示此变量的string字符串
  override string toString() { result = this.getType().getName() + " " + this.getName() }

  /** Gets the initializer expression of this local variable declaration. */
  // 获取此局部变量声明的初始化表达式。 例如：String code = request.getParameter("code");得到request.getParameter("code");
  override Expr getInitializer() { result = getDeclExpr().getInit() }
 
  // CodeQL当前类名
  override string getAPrimaryQlClass() { result = "LocalVariableDecl" }
}
```

### 4 Parameter类
```ql
// 可调用形参
class Parameter extends Element, @param, LocalScopeVariable {
  // 形参参数类型
  override Type getType() { params(this, result, _, _, _) }

  // 形参没有在可调用内部没有被重新赋值。getAnAssignedValue(): 获取变量右表达式
  predicate isEffectivelyFinal() { not exists(getAnAssignedValue()) }

  // 获取该形参（从零开始的）索引（可调用对象）。
  int getPosition() { params(this, _, result, _, _) }

  // 获取声明此形参的可调用对象。对于Parameter来说，获取的是该参数所在的函数
  override Callable getCallable() { params(this, _, _, result, _) }

  // 获取这个形参的源
  Parameter getSourceDeclaration() { params(this, _, _, _, result) }

  // 形参本身是源
  predicate isSourceDeclaration() { this.getSourceDeclaration() = this }

  // 形参是可变参数。例如：private boolean isValid(String... name) {...}; `String... name`为可变参数，可以看Java对可变参数的定义
  predicate isVarargs() { isVarargsParam(this) }

  // 获取形参可调用对象被调用处的参数。如：boolean result = isValid(name); ...; private boolean isValid(String name) {...}; 函数，形参为String name，此处获取调用isValid函数处的name参数
  // 此处形参不能是可变参数
  Expr getAnArgument() {
    not isVarargs() and
    result = getACallArgument(getPosition())
  }

  pragma[noinline]
  private Expr getACallArgument(int i) {
    exists(Call call |
      result = call.getArgument(i) and // 可调用对象调用处指定位置的参数
      call.getCallee().getSourceDeclaration().getAParameter() = this  // 调用目标的可调用对象来源的形参
    )
  }
  
   // CodeQL当前类名
  override string getAPrimaryQlClass() { result = "Parameter" }
}
```
