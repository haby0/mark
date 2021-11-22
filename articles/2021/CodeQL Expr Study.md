codeql\java\ql\src\semmle\code\java\Expr.qll

CodeQL对Java表达式的处理

### 1 Expr类
```ql
/** A common super-class that represents all kinds of expressions. */
// 表达式类建模
class Expr extends ExprParent, @expr {
  // 表达式表示形式
  /*abstract*/ override string toString() { result = "expr" }

  /**
   * Gets the callable in which this expression occurs, if any.
   */
  // 获取表达式的可调用对象。可调用对象：方法或构造函数 
  Callable getEnclosingCallable() { callableEnclosingExpr(this, result) }

  /** Gets the index of this expression as a child of its parent. */
  // 获取该表达式在父表达式中的索引
  int getIndex() { exprs(this, _, _, _, result) }

  /** Gets the parent of this expression. */
  // 父表达式
  ExprParent getParent() { exprs(this, _, _, result, _) }

  /** Holds if this expression is the child of the specified parent at the specified (zero-based) position. */
  // 表达式是父表达式指定位置则成立
  predicate isNthChildOf(ExprParent parent, int index) { exprs(this, _, _, parent, index) }

  /** Gets the type of this expression. */
  // 表达式类型
  Type getType() { exprs(this, _, result, _, _) }

  /** Gets the compilation unit in which this expression occurs. */
  // 获取次表达式的编译单元（表达式所在文件）
  CompilationUnit getCompilationUnit() { result = this.getFile() }

  /**
   * Gets the kind of this expression.
   *
   * Each kind of expression has a unique (integer) identifier.
   * This is an implementation detail that should not normally
   * be referred to by library users, since the kind of an expression
   * is also represented by its QL type.
   *
   * In a few rare situations, referring to the kind of an expression
   * via its unique identifier may be appropriate; for example, when
   * comparing whether two expressions have the same kind (as opposed
   * to checking whether an expression has a particular kind).
   */
  // 获取表达式类型
  int getKind() { exprs(this, result, _, _, _) }
  
  /**
   * DEPRECATED: This is no longer necessary. See `Expr.isParenthesized()`.
   *
   * Gets this expression with any surrounding parentheses removed.
   */
  // 废弃
  // 获取去除括号或的表达式
  deprecated Expr getProperExpr() {
    result = this.(ParExpr).getExpr().getProperExpr()
    or
    result = this and not this instanceof ParExpr
  }

  /** Gets the statement containing this expression, if any. */
  // 获取包含此表达式的语句
  Stmt getEnclosingStmt() { statementEnclosingExpr(this, result) }

  /**
   * Gets a statement that directly or transitively contains this expression, if any.
   * This is equivalent to `this.getEnclosingStmt().getEnclosingStmt*()`.
   */
  // 获取包含该表达式的所有语句
  Stmt getAnEnclosingStmt() { result = this.getEnclosingStmt().getEnclosingStmt*() }

  /** Gets a child of this expression. */
  // 获取子表达式
  Expr getAChildExpr() { exprs(result, _, _, this, _) }

  /** Gets the basic block in which this expression occurs, if any. */
  // 获取出现该表达式的基本块。基本块继承控制流节点。
  BasicBlock getBasicBlock() { result.getANode() = this }

  /** Gets the `ControlFlowNode` corresponding to this expression. */
  // 当前表达式也是控制流节点，返回控制流节点形式
  ControlFlowNode getControlFlowNode() { result = this }

  /** This statement's Halstead ID (used to compute Halstead metrics). */
  // 此语句的 Halstead ID（用于计算 Halstead 指标）
  string getHalsteadID() { result = this.toString() }

  /**
   * Holds if this expression is a compile-time constant.
   *
   * See JLS v8, section 15.28 (Constant Expressions).
   */
  // 表达式是否是编译时常量。参考：https://docs.oracle.com/javase/specs/jls/se8/jls8.pdf
  predicate isCompileTimeConstant() { this instanceof CompileTimeConstantExpr }

  /** Holds if this expression occurs in a static context. */
  // 表达式在静态上下文中则成立
  predicate isInStaticContext() {
    /*
     * JLS 8.1.3 (Inner Classes and Enclosing Instances)
     * A statement or expression occurs in a static context if and only if the
     * innermost method, constructor, instance initializer, static initializer,
     * field initializer, or explicit constructor invocation statement
     * enclosing the statement or expression is a static method, a static
     * initializer, the variable initializer of a static variable, or an
     * explicit constructor invocation statement.
     */

    this.getEnclosingCallable().isStatic()  // 表达式可调用对象是静态
    or
    this.getParent+() instanceof ThisConstructorInvocationStmt  // 父表达式是调用构造函数。例如：this(test);  test是表达式
    or
    this.getParent+() instanceof SuperConstructorInvocationStmt  // 父表达式是调用父类构造函数。例如：super(test);  test是表达式
    or
    exists(LambdaExpr lam |  // Lambda表达式
      lam.asMethod() = this.getEnclosingCallable() and lam.isInStaticContext()
    )
  }

  /** Holds if this expression is parenthesized. */
  // 表达式是被括号括起来的
  predicate isParenthesized() { isParenthesized(this, _) }
}
```
