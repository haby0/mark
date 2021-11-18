codeql\java\ql\src\semmle\code\java\Variable.qll

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