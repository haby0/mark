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

### 4 SsaSourceField类

```qll
class SsaSourceField extends SsaSourceVariable {
  // 继承SsaSourceVariable，得到除局部作用域变量外的变量
  SsaSourceField() {
    this = TPlainField(_, _) or this = TEnclosingField(_, _, _) or this = TQualifiedField(_, _, _)
  }

  /** Gets the field corresponding to this named field. */
  // 调用父类`SsaSourceVariable`的`getVariable`方法获取`SsaSourceField`类型字段
  Field getField() { result = getVariable() }

  /** Gets a string representation of the qualifier. */
  // 获取限定符的字符串表示形式
  string ppQualifier() {
    // 静态字段返回全限定类名，否则返回this
    exists(Field f | this = TPlainField(_, f) |
      if f.isStatic() then result = f.getDeclaringType().getQualifiedName() else result = "this"
    )
    or
    // 外部类名 + this
    exists(Field f, RefType t | this = TEnclosingField(_, f, t) | result = t.toString() + ".this")
    or
    // SsaSourceVariable类型变量的限定符字符串
    exists(SsaSourceVariable q | this = TQualifiedField(_, q, _) | result = q.toString())
  }

  /** Holds if the field itself or any of the fields part of the qualifier are volatile. */
  // 如果字段本身或限定符的任何字段部分使用了`volatile`关键字，则成立
  predicate isVolatile() {
    getField().isVolatile() or
    getQualifier().(SsaSourceField).isVolatile()
  }
}
```

### 5 TrackedVariablesImpl模块

```ql
module TrackedVariablesImpl {
  // 字段访问数量
  private int numberOfAccesses(SsaSourceField f) {
    result = strictcount(FieldAccess fa | fa = f.getAnAccess())
  }

  // 三个条件满足一个即可
  private predicate loopAccessed(SsaSourceField f) {
    exists(LoopStmt l, FieldRead fr | fr = f.getAnAccess() | // LoopStmt:循环语句
      l.getBody() = fr.getEnclosingStmt().getEnclosingStmt*() or // 字段访问在循环语句中。fr.getEnclosingStmt():获取包含字段访问的表达式语句
      l.getCondition() = fr.getParent*() or // 循环语句布尔条件表达式中包含字段访问。l.getCondition():循环语句的布尔条件
      l.(ForStmt).getAnUpdate() = fr.getParent*() // for循环语句的更新表达式中包含字段访问。l.(ForStmt).getAnUpdate():for循环语句的更新表达式
    )
  }

  // SsaSourceField字段在循环语句中 或 SsaSourceField字段访问次数大于1次
  private predicate multiAccessed(SsaSourceField f) { loopAccessed(f) or 1 < numberOfAccesses(f) }

  // （SsaSourceField字段在循环语句中 或 SsaSourceField字段访问次数大于1次）并且SsaSourceField字段没有（使用了`volatile`关键字或限定符使用了`volatile`关键字）
  cached
  predicate trackField(SsaSourceField f) { multiAccessed(f) and not f.isVolatile() }

  // 跟踪变量
  class TrackedVar extends SsaSourceVariable {
    TrackedVar() {
      this = TLocalVar(_, _) or //局部作用域变量或变量访问在函数或构造函数中
      trackField(this) //SsaSourceVariable变量不是volatile变量并且（在循环语句中或访问次数大于1次）
    }
  }

  // 跟踪字段。SsaSourceField字段或跟踪变量字段
  class TrackedField extends TrackedVar, SsaSourceField { }
}
```

### 6 SsaImpl模块

```ql
cached
private module SsaImpl {
  cached
  TrackedVar getDestVar(VariableUpdate upd) {
    result.getAnAccess() = upd.(Assignment).getDest()
    or
    exists(LocalVariableDecl v | v = upd.(LocalVariableDeclExpr).getVariable() |
      result = TLocalVar(v.getCallable(), v)
    )
    or
    result.getAnAccess() = upd.(UnaryAssignExpr).getExpr()
  }

  cached
  predicate certainVariableUpdate(TrackedVar v, ControlFlowNode n, BasicBlock b, int i) {
    exists(VariableUpdate a | a = n | getDestVar(a) = v) and
    b.getNode(i) = n and
    hasDominanceInformation(b)
    or
    certainVariableUpdate(v.getQualifier(), n, b, i)
  }

  private ControlFlowNode parentDef(NestedClass nc) {
    nc.(AnonymousClass).getClassInstanceExpr() = result or
    nc.(LocalClass).getLocalClassDeclStmt() = result
  }

  private RefType desugaredGetEnclosingType(NestedClass inner) {
    exists(ControlFlowNode node |
      node = parentDef(inner) and
      node.getEnclosingCallable().getDeclaringType() = result
    )
  }

  private ControlFlowNode captureNode(TrackedVar capturedvar, TrackedVar closurevar) {
    exists(
      LocalScopeVariable v, Callable inner, Callable outer, NestedClass innerclass, VarAccess va
    |
      va.getVariable() = v and
      inner = va.getEnclosingCallable() and
      outer = v.getCallable() and
      inner != outer and
      inner.getDeclaringType() = innerclass and
      result = parentDef(desugaredGetEnclosingType*(innerclass)) and
      result.getEnclosingStmt().getEnclosingCallable() = outer and
      capturedvar = TLocalVar(outer, v) and
      closurevar = TLocalVar(inner, v)
    )
  }

  private predicate variableUse(TrackedVar v, RValue use, BasicBlock b, int i) {
    v.getAnAccess() = use and b.getNode(i) = use
  }

  private predicate variableCapture(
    TrackedVar capturedvar, TrackedVar closurevar, BasicBlock b, int i
  ) {
    b.getNode(i) = captureNode(capturedvar, closurevar)
  }

  private predicate variableUseOrCapture(TrackedVar v, BasicBlock b, int i) {
    variableUse(v, _, b, i) or variableCapture(v, _, b, i)
  }

  private predicate liveAtEntry(TrackedVar v, BasicBlock b) {
    exists(int i | variableUseOrCapture(v, b, i) |
      not exists(int j | certainVariableUpdate(v, _, b, j) | j < i)
    )
    or
    liveAtExit(v, b) and not certainVariableUpdate(v, _, b, _)
  }

  private predicate liveAtExit(TrackedVar v, BasicBlock b) { liveAtEntry(v, b.getABBSuccessor()) }

  private predicate init(FieldWrite fw) {
    fw.getEnclosingCallable() instanceof InitializerMethod
    or
    fw.getEnclosingCallable() instanceof Constructor and fw.isOwnFieldAccess()
    or
    exists(LocalVariableDecl v |
      v.getAnAccess() = fw.getQualifier() and
      forex(VariableAssign va | va.getDestVar() = v and exists(va.getSource()) |
        va.getSource() instanceof ClassInstanceExpr
      )
    )
  }
  
  cached
  predicate relevantFieldUpdate(Callable c, Field f, FieldWrite fw) {
    fw = f.getAnAccess() and
    not init(fw) and
    fw.getEnclosingCallable() = c and
    exists(TrackedField nf | nf.getField() = f)
  }

  private predicate setsOwnField(Method c, Field f) {
    exists(FieldWrite fw | relevantFieldUpdate(c, f, fw) and fw.isOwnFieldAccess())
  }

  private predicate setsOtherField(Callable c, Field f) {
    exists(FieldWrite fw | relevantFieldUpdate(c, f, fw) and not fw.isOwnFieldAccess())
  }

  pragma[nomagic]
  private predicate innerclassSupertypeStar(InnerClass t1, RefType t2) {
    t1.getASupertype*().getSourceDeclaration() = t2
  }

  private predicate intraInstanceCallEdge(Callable c1, Method m2) {
    exists(MethodAccess ma, RefType t1 |
      ma.getCaller() = c1 and
      m2 = viableImpl(ma) and
      not m2.isStatic() and
      (
        not exists(ma.getQualifier()) or
        ma.getQualifier() instanceof ThisAccess or
        ma.getQualifier() instanceof SuperAccess
      ) and
      c1.getDeclaringType() = t1 and
      if t1 instanceof InnerClass
      then
        innerclassSupertypeStar(t1, ma.getCallee().getSourceDeclaration().getDeclaringType()) and
        not exists(ma.getQualifier().(ThisAccess).getQualifier()) and
        not exists(ma.getQualifier().(SuperAccess).getQualifier())
      else any()
    )
  }

  private Callable tgt(Call c) {
    result = viableImpl(c)
    or
    result = getRunnerTarget(c)
    or
    c instanceof ConstructorCall and result = c.getCallee().getSourceDeclaration()
  }

  private predicate callEdge(Callable c1, Callable c2) {
    exists(Call c | c.getCaller() = c1 and c2 = tgt(c))
  }

  private predicate crossInstanceCallEdge(Callable c1, Callable c2) {
    callEdge(c1, c2) and not intraInstanceCallEdge(c1, c2)
  }

  private predicate updateCandidate(TrackedField f, Call call, BasicBlock b, int i) {
    b.getNode(i) = call and
    call.getEnclosingCallable() = f.getEnclosingCallable() and
    relevantFieldUpdate(_, f.getField(), _)
  }

  private predicate callDefUseRank(TrackedField f, BasicBlock b, int rankix, int i) {
    updateCandidate(f, _, b, _) and
    i =
      rank[rankix](int j |
        certainVariableUpdate(f, _, b, j) or
        variableUseOrCapture(f, b, j) or
        updateCandidate(f, _, b, j)
      )
  }

  private predicate liveAtRank(TrackedField f, BasicBlock b, int rankix, int i) {
    callDefUseRank(f, b, rankix, i) and
    (
      rankix = max(int rix | callDefUseRank(f, b, rix, _)) and liveAtExit(f, b)
      or
      variableUseOrCapture(f, b, i)
      or
      exists(int j | liveAtRank(f, b, rankix + 1, j) and not certainVariableUpdate(f, _, b, j))
    )
  }

  private predicate relevantCall(Call call, TrackedField f) {
    exists(BasicBlock b, int i |
      updateCandidate(f, call, b, i) and
      liveAtRank(f, b, _, i)
    )
  }

  private predicate source(Call call, TrackedField f, Field field, Callable c, boolean fresh) {
    relevantCall(call, f) and
    field = f.getField() and
    c = tgt(call) and
    if c instanceof Constructor then fresh = true else fresh = false
  }

  private newtype TCallableNode =
    MkCallableNode(Callable c, boolean fresh) { source(_, _, _, c, fresh) or edge(_, c, fresh) }

  private predicate edge(TCallableNode n, Callable c2, boolean f2) {
    exists(Callable c1, boolean f1 | n = MkCallableNode(c1, f1) |
      intraInstanceCallEdge(c1, c2) and f2 = f1
      or
      crossInstanceCallEdge(c1, c2) and
      if c2 instanceof Constructor then f2 = true else f2 = false
    )
  }

  private predicate edge(TCallableNode n1, TCallableNode n2) {
    exists(Callable c2, boolean f2 |
      edge(n1, c2, f2) and
      n2 = MkCallableNode(c2, f2)
    )
  }

  pragma[noinline]
  private predicate source(Call call, TrackedField f, Field field, TCallableNode n) {
    exists(Callable c, boolean fresh |
      source(call, f, field, c, fresh) and
      n = MkCallableNode(c, fresh)
    )
  }

  private predicate sink(Callable c, Field f, TCallableNode n) {
    setsOwnField(c, f) and n = MkCallableNode(c, false)
    or
    setsOtherField(c, f) and n = MkCallableNode(c, _)
  }

  private predicate prunedNode(TCallableNode n) {
    sink(_, _, n)
    or
    exists(TCallableNode mid | edge(n, mid) and prunedNode(mid))
  }

  private predicate prunedEdge(TCallableNode n1, TCallableNode n2) {
    prunedNode(n1) and
    prunedNode(n2) and
    edge(n1, n2)
  }

  private predicate edgePlus(TCallableNode c1, TCallableNode c2) = fastTC(prunedEdge/2)(c1, c2)

  pragma[noopt]
  cached
  predicate updatesNamedField(Call call, TrackedField f, Callable setter) {
    exists(TCallableNode src, TCallableNode sink, Field field |
      source(call, f, field, src) and
      sink(setter, field, sink) and
      (src = sink or edgePlus(src, sink))
    )
  }

  cached
  predicate uncertainVariableUpdate(TrackedVar v, ControlFlowNode n, BasicBlock b, int i) {
    exists(Call c | c = n | updatesNamedField(c, v, _)) and
    b.getNode(i) = n and
    hasDominanceInformation(b)
    or
    uncertainVariableUpdate(v.getQualifier(), n, b, i)
  }

  private predicate variableUpdate(TrackedVar v, ControlFlowNode n, BasicBlock b, int i) {
    certainVariableUpdate(v, n, b, i) or uncertainVariableUpdate(v, n, b, i)
  }

  cached
  predicate phiNode(TrackedVar v, BasicBlock b) {
    liveAtEntry(v, b) and
    exists(BasicBlock def | dominanceFrontier(def, b) |
      variableUpdate(v, _, def, _) or phiNode(v, def)
    )
  }

  cached
  predicate hasEntryDef(TrackedVar v, BasicBlock b) {
    exists(LocalScopeVariable l, Callable c | v = TLocalVar(c, l) and c.getBody() = b |
      l instanceof Parameter or
      l.getCallable() != c
    )
    or
    v instanceof SsaSourceField and v.getEnclosingCallable().getBody() = b and liveAtEntry(v, b)
  }

  cached
  module SsaDefReaches {
    private predicate defUseRank(TrackedVar v, BasicBlock b, int rankix, int i) {
      i =
        rank[rankix](int j |
          any(TrackedSsaDef def).definesAt(v, b, j) or variableUseOrCapture(v, b, j)
        )
    }

    private int lastRank(TrackedVar v, BasicBlock b) {
      result = max(int rankix | defUseRank(v, b, rankix, _))
    }

    private predicate ssaDefRank(TrackedVar v, TrackedSsaDef def, BasicBlock b, int rankix) {
      exists(int i |
        def.definesAt(v, b, i) and
        defUseRank(v, b, rankix, i)
      )
    }

    private predicate ssaDefReachesRank(TrackedVar v, TrackedSsaDef def, BasicBlock b, int rankix) {
      ssaDefRank(v, def, b, rankix)
      or
      ssaDefReachesRank(v, def, b, rankix - 1) and
      rankix <= lastRank(v, b) and
      not ssaDefRank(v, _, b, rankix)
    }

    cached
    predicate ssaDefReachesEndOfBlock(TrackedVar v, TrackedSsaDef def, BasicBlock b) {
      liveAtExit(v, b) and
      (
        ssaDefReachesRank(v, def, b, lastRank(v, b))
        or
        exists(BasicBlock idom |
          bbIDominates(idom, b) and // It is sufficient to traverse the dominator graph, cf. discussion above.
          ssaDefReachesEndOfBlock(v, def, idom) and
          not any(TrackedSsaDef other).definesAt(v, b, _)
        )
      )
    }

    private predicate ssaDefReachesUseWithinBlock(TrackedVar v, TrackedSsaDef def, RValue use) {
      exists(BasicBlock b, int rankix, int i |
        ssaDefReachesRank(v, def, b, rankix) and
        defUseRank(v, b, rankix, i) and
        variableUse(v, use, b, i)
      )
    }

    cached
    predicate ssaDefReachesUse(TrackedVar v, TrackedSsaDef def, RValue use) {
      ssaDefReachesUseWithinBlock(v, def, use)
      or
      exists(BasicBlock b |
        variableUse(v, use, b, _) and
        ssaDefReachesEndOfBlock(v, def, b.getABBPredecessor()) and
        not ssaDefReachesUseWithinBlock(v, _, use)
      )
    }

    private predicate ssaDefReachesCaptureWithinBlock(
      TrackedVar v, TrackedSsaDef def, TrackedVar closurevar
    ) {
      exists(BasicBlock b, int rankix, int i |
        ssaDefReachesRank(v, def, b, rankix) and
        defUseRank(v, b, rankix, i) and
        variableCapture(v, closurevar, b, i)
      )
    }

    cached
    predicate ssaDefReachesCapture(TrackedVar v, TrackedSsaDef def, TrackedVar closurevar) {
      ssaDefReachesCaptureWithinBlock(v, def, closurevar)
      or
      exists(BasicBlock b |
        variableCapture(v, closurevar, b, _) and
        ssaDefReachesEndOfBlock(v, def, b.getABBPredecessor()) and
        not ssaDefReachesCaptureWithinBlock(v, _, closurevar)
      )
    }

    private predicate ssaDefReachesUncertainDefWithinBlock(
      TrackedVar v, TrackedSsaDef def, SsaUncertainImplicitUpdate redef
    ) {
      exists(BasicBlock b, int rankix, int i |
        ssaDefReachesRank(v, def, b, rankix) and
        defUseRank(v, b, rankix + 1, i) and
        redef.(TrackedSsaDef).definesAt(v, b, i)
      )
    }

    cached
    predicate ssaDefReachesUncertainDef(
      TrackedVar v, TrackedSsaDef def, SsaUncertainImplicitUpdate redef
    ) {
      ssaDefReachesUncertainDefWithinBlock(v, def, redef)
      or
      exists(BasicBlock b |
        redef.(TrackedSsaDef).definesAt(v, b, _) and
        ssaDefReachesEndOfBlock(v, def, b.getABBPredecessor()) and
        not ssaDefReachesUncertainDefWithinBlock(v, _, redef)
      )
    }
  }
  
  private module AdjacentUsesImpl {
    private predicate defUseRank(TrackedVar v, BasicBlock b, int rankix, int i) {
      i = rank[rankix](int j | any(TrackedSsaDef def).definesAt(v, b, j) or variableUse(v, _, b, j))
    }
    
    private int lastRank(TrackedVar v, BasicBlock b) {
      result = max(int rankix | defUseRank(v, b, rankix, _))
    }

    private predicate varOccursInBlock(TrackedVar v, BasicBlock b) { defUseRank(v, b, _, _) }

    private predicate blockPrecedesVar(TrackedVar v, BasicBlock b) {
      varOccursInBlock(v, b)
      or
      ssaDefReachesEndOfBlock(v, _, b)
    }

    private predicate varBlockReaches(TrackedVar v, BasicBlock b1, BasicBlock b2) {
      varOccursInBlock(v, b1) and
      b2 = b1.getABBSuccessor() and
      blockPrecedesVar(v, b2)
      or
      exists(BasicBlock mid |
        varBlockReaches(v, b1, mid) and
        b2 = mid.getABBSuccessor() and
        not varOccursInBlock(v, mid) and
        blockPrecedesVar(v, b2)
      )
    }

    private predicate varBlockStep(TrackedVar v, BasicBlock b1, BasicBlock b2) {
      varBlockReaches(v, b1, b2) and
      varOccursInBlock(v, b2)
    }

    pragma[nomagic]
    predicate adjacentVarRefs(TrackedVar v, BasicBlock b1, int i1, BasicBlock b2, int i2) {
      exists(int rankix |
        b1 = b2 and
        defUseRank(v, b1, rankix, i1) and
        defUseRank(v, b2, rankix + 1, i2)
      )
      or
      defUseRank(v, b1, lastRank(v, b1), i1) and
      varBlockStep(v, b1, b2) and
      defUseRank(v, b2, 1, i2)
    }
  }

  private import AdjacentUsesImpl

  cached
  predicate firstUse(TrackedSsaDef def, RValue use) {
    exists(TrackedVar v, BasicBlock b1, int i1, BasicBlock b2, int i2 |
      adjacentVarRefs(v, b1, i1, b2, i2) and
      def.definesAt(v, b1, i1) and
      variableUse(v, use, b2, i2)
    )
    or
    exists(TrackedVar v, TrackedSsaDef redef, BasicBlock b1, int i1, BasicBlock b2, int i2 |
      redef instanceof SsaUncertainImplicitUpdate or redef instanceof SsaPhiNode
    |
      adjacentVarRefs(v, b1, i1, b2, i2) and
      def.definesAt(v, b1, i1) and
      redef.definesAt(v, b2, i2) and
      firstUse(redef, use)
    )
  }

  cached
  module SsaPublic {
    cached
    predicate adjacentUseUseSameVar(RValue use1, RValue use2) {
      exists(TrackedVar v, BasicBlock b1, int i1, BasicBlock b2, int i2 |
        adjacentVarRefs(v, b1, i1, b2, i2) and
        variableUse(v, use1, b1, i1) and
        variableUse(v, use2, b2, i2)
      )
    }

    cached
    predicate adjacentUseUse(RValue use1, RValue use2) {
      adjacentUseUseSameVar(use1, use2)
      or
      exists(TrackedVar v, TrackedSsaDef def, BasicBlock b1, int i1, BasicBlock b2, int i2 |
        adjacentVarRefs(v, b1, i1, b2, i2) and
        variableUse(v, use1, b1, i1) and
        def.definesAt(v, b2, i2) and
        firstUse(def, use2) and
        (def instanceof SsaUncertainImplicitUpdate or def instanceof SsaPhiNode)
      )
    }
  }
}
```
