codeql\java\ql\src\semmle\code\java\Expr.qll

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
  // 获取表达式的可调用对象 
  Callable getEnclosingCallable() { callableEnclosingExpr(this, result) }

  /** Gets the index of this expression as a child of its parent. */
  // 获取该表达式在父表达式中的索引
  int getIndex() { exprs(this, _, _, _, result) }

  /** Gets the parent of this expression. */
  ExprParent getParent() { exprs(this, _, _, result, _) }

  /** Holds if this expression is the child of the specified parent at the specified (zero-based) position. */
  predicate isNthChildOf(ExprParent parent, int index) { exprs(this, _, _, parent, index) }

  /** Gets the type of this expression. */
  Type getType() { exprs(this, _, result, _, _) }

  /** Gets the compilation unit in which this expression occurs. */
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
  int getKind() { exprs(this, result, _, _, _) }

  /** Gets the statement containing this expression, if any. */
  Stmt getEnclosingStmt() { statementEnclosingExpr(this, result) }

  /**
   * Gets a statement that directly or transitively contains this expression, if any.
   * This is equivalent to `this.getEnclosingStmt().getEnclosingStmt*()`.
   */
  Stmt getAnEnclosingStmt() { result = this.getEnclosingStmt().getEnclosingStmt*() }

  /** Gets a child of this expression. */
  Expr getAChildExpr() { exprs(result, _, _, this, _) }

  /** Gets the basic block in which this expression occurs, if any. */
  BasicBlock getBasicBlock() { result.getANode() = this }

  /** Gets the `ControlFlowNode` corresponding to this expression. */
  ControlFlowNode getControlFlowNode() { result = this }

  /** This statement's Halstead ID (used to compute Halstead metrics). */
  string getHalsteadID() { result = this.toString() }

  /**
   * Holds if this expression is a compile-time constant.
   *
   * See JLS v8, section 15.28 (Constant Expressions).
   */
  predicate isCompileTimeConstant() { this instanceof CompileTimeConstantExpr }

  /** Holds if this expression occurs in a static context. */
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

    this.getEnclosingCallable().isStatic()
    or
    this.getParent+() instanceof ThisConstructorInvocationStmt
    or
    this.getParent+() instanceof SuperConstructorInvocationStmt
    or
    exists(LambdaExpr lam |
      lam.asMethod() = this.getEnclosingCallable() and lam.isInStaticContext()
    )
  }

  /** Holds if this expression is parenthesized. */
  predicate isParenthesized() { isParenthesized(this, _) }
}
```
