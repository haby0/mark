文件路径: codeql\java\ql\lib\semmle\code\java\ControlFlowGraph.qll

建模用于计算表达式级过程内控制流图的类和谓词


### 1 ControlFlowNode 类


前驱和后继：
设结点 A1 和结点 A2 有一条有向边，则 A1 是 A2 的前驱，A2 是 A1 的后继

```qll
@top = @element | @locatable | @folder;
@element = @file | @package | @primitive | @class | @interface | @method | @constructor | @modifier | @param | @exception | @field |
           @annotation | @boundedtype | @array | @localvar | @expr | @stmt | @import | @fielddecl;
@locatable = @file | @class | @interface | @fielddecl | @field | @constructor | @method | @param | @exception
            | @boundedtype | @typebound | @array | @primitive
            | @import | @stmt | @expr | @localvar | @javadoc | @javadocTag | @javadocText
            | @xmllocatable;

class Top extends @top {}            

@exprparent = @stmt | @expr | @callable | @field | @fielddecl | @class | @interface | @param | @localvar | @typevariable;


class ControlFlowNode extends Top, @exprparent {
  /** Gets the statement containing this node, if any. */
  // 获取包含此节点的语句
  Stmt getEnclosingStmt() {
    result = this or
    result = this.(Expr).getEnclosingStmt()
  }

  /** Gets the immediately enclosing callable whose body contains this node. */
  // 获取其主体包含此节点的直接封闭可调用对象
  Callable getEnclosingCallable() {
    result = this or
    result = this.(Stmt).getEnclosingCallable() or
    result = this.(Expr).getEnclosingCallable()
  }

  /** Gets an immediate successor of this node. */
  // 获取此节点的直接后继节点
  ControlFlowNode getASuccessor() { result = succ(this) }

  /** Gets an immediate predecessor of this node. */
  // 获取此节点的直接前驱节点
  ControlFlowNode getAPredecessor() { this = succ(result) }

  /** Gets an exception successor of this node. */
  // 获取此节点的异常后继节点（通过ThrowCompletion得到）
  ControlFlowNode getAnExceptionSuccessor() { result = succ(this, ThrowCompletion(_)) }

  /** Gets a successor of this node that is neither an exception successor nor a jump (break, continue, return). */
  // 获取此节点的后继节点，既不是异常后继节点也不是一个跳转（break, continue, return）
  ControlFlowNode getANormalSuccessor() {
    result = succ(this, BooleanCompletion(_, _)) or
    result = succ(this, NormalCompletion())
  }

  /** Gets the intra-procedural successor of `n`. */
  // 获取包含该节点的基本块（BB）
  BasicBlock getBasicBlock() { result.getANode() = this }
}
```
