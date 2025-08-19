# TermCraft: 形式系统解谜游戏 - 玩家与关卡设计手册

> 目标读者：新玩家 + 希望利用**开发者模式**创造、分享和理解复杂关卡的关卡设计师 / 高级用户。  
> 本手册先给玩家易读的玩法说明（第 1、2 节），然后以技术细节（第 3 节）完整呈现引擎行为、语法和设计注意事项，方便你精确地构造可解的、富有表现力的关卡。

---

## 1. 游戏概述

**一句话简介**  
TermCraft 是一个把**形式项（term）** 视作树状结构，并通过**重写规则（rewrite）与等价公理（equiv）** 进行变换的解谜游戏 — 你的任务是把“当前项 (Current Term)”一步步变成“目标项 (Target Term)”。

**游戏目标**  
玩家通过执行重写动作或在界面上的占位符处**构造**新项（消耗预算），以在有限的燃料（Fuel）与预算（Budget）下把 `Current Term` 变换为 `Target Term`。当两者**完全相等**（结构与常量一致）即获胜。

---

## 2. 核心玩法与机制

### 2.1 理解“项 (Term)”

- 在 TermCraft 中，“项”是程序内部的主数据类型：它是**树状结构**，节点有三种基本类型：
    
    1. **常量 (Constant)** — 原子名称，例如 `apple`、`red`。（在代码中为 `Term.Const(name)`）
        
    2. **变量 (Variable)** — 形式为 `var(X)`，表示可被模式匹配绑定的占位符（在代码中为 `Term.Var(name)`）。注意：写法为 `var(<name>)`，`var` 关键字在解析时不区分大小写。
        
    3. **复合项 (Compound Term)** — 形如 `name(arg1,arg2,...)`，用来表示结构化数据或构造子（在代码中为 `Term.Compound(name, children)`）。
        

**例子**

- 常量：`apple`
    
- 变量：`var(X)`
    
- 复合项：`pair(apple,green)`、`rev_pair(var(Y), var(X))`、`puzzle_state(...)`
    

> 概念上：项 = 节点类型（CONST / VAR / COMPOUND），复合项有构造子名称（constructor）与子项数组（children）。

---

### 2.2 关键资源

**燃料 (Fuel)**

- 表示**可执行重写动作**的次数。每次玩家选择并实际应用一条 `rewrite` 规则（把匹配处替换为 RHS 并把变量代入）都会消耗 **1 点燃料**。当燃料为 0 且无法采取其它行动时玩家会失败。
    

**预算 (Budget)**

- 用于“构造 (Construct)”动作：将 UI 中的 `placeholder` 常量替换为玩家输入的新项。构造新项的**代价 = 项的 Size**（详见第 3.4）。每次构造会从预算中减去新项的代价。预算不能为负；构造时 UI 会阻止代价超过剩余预算的操作。
    

---

### 2.3 玩家的动作

**动作一：重写 (Rewrite)**

- 概念：引擎扫描当前项树中所有 `rewrite(LHS, RHS)` 的子项，把它们视为可选规则；当某处**地面子项**（不含变量）能与 `LHS` 匹配（考虑等价公理时匹配更宽），玩家可以选择把该处替换为 `RHS`（RHS 中的变量由匹配得到的替换值代入）。
    
- 代价：**消耗 1 点燃料**（Fuel）。
    
- 限制（由引擎强制）：
    
    - `rewrite` 必须**二元**（arity = 2）。
        
    - 引擎在收集规则时**会拒绝**那些在 RHS 引入任何 `LHS` 没有绑定的游离变量（RHS 不得引入未在 LHS 出现的变量）——这是不合法规则，会被忽略。
        
- 匹配细节：匹配通过 **合一（syntactic matching）** 完成，但先对被匹配位置应用“同余闭包 / 等价公理（equiv）”的扩展 —— 即引擎会把目标位置的等价类中的每一个代表项都试图与 LHS 做语法匹配，从而发现因为 `equiv` 导致的额外匹配点。
    
- 示例规则：  
    `rewrite(pair(var(X), var(Y)), rev_pair(var(Y), var(X)))`  
    当 `pair(apple, green)` 在场且 `pair` 与 `pair` 匹配时，可以得到 `rev_pair(green, apple)`（消耗 1 燃料）。
    

**动作二：构造 (Construct)**

- 概念：界面上某些常量形态的 `placeholder` 会被标示为可点击（黄色高亮）。玩家点击并在弹出模态中输入一个新项串，新项会替换该 `placeholder`。
    
- 代价：新项的代价由 `Size(newItem)` 决定；代价必须 **≤ 剩余 Budget**。构造会从预算中扣除该值。
    
- 限制（由引擎强制）：
    
    - 只允许构造**地面项**（不含变量）。如果输入含变量，构造会被拒绝。
        
    - UI 在提交时会做格式校验，错误会在模态中提示（并阻止提交）。
        

---

### 2.4 胜利与失败

- **胜利条件**：当 `Current Term`（游戏内的 `GameState.T`）与 `Target Term`（`TargetTerm`）结构与常量完全相同（`Term.equals` 比较 `toString()`），即触发胜利模态。
    
- **失败条件**：若无法继续（燃料耗尽且没有构造或可用重写），或无任何有效动作可用，会触发失败模态并显示原因。
    

---

## 3. 开发者模式与关卡设计指南

> 本节基于对游戏源代码的审计（特别是 `Term.fromString`, `FindValidRewrites`, `CalculateEquivalenceRelation`, `FindPlaceholderSites`, `Size` 等函数）。设计关卡时，请以此节为行为规范。

---

### 3.1 开启方式与面板概览

**如何开启开发者模式**

- 页面右上角有按钮 `开发者模式`（id=`dev-toggle-button`）。点击可切换开发者面板（id=`dev-panel`）。
    
- 打开面板时，会自动由当前 `GameState` 填充字段；点击 `应用设置` 会尝试解析输入并重新启动游戏循环（`MainGameLoop`）。
    

**面板字段（表格）**

|显示/输入项|HTML id|说明|
|---|--:|---|
|开关按钮|`dev-toggle-button`|切换开发者面板显示/隐藏|
|开发者面板根|`dev-panel`|包含所有开发者输入项（初始为 `hidden`）|
|燃料输入|`dev-fuel-input`|设置初始 Fuel（非负整数）|
|预算输入|`dev-budget-input`|设置初始 Budget（非负整数）|
|当前项编辑|`dev-current-term-input`|多行文本；支持换行，提交前会移除所有空白字符（`replace(/\s/g,'')`）再解析|
|目标项编辑|`dev-target-term-input`|多行文本；同上|
|错误消息|`dev-error-message`|显示解析或应用错误|
|应用按钮|`dev-apply-button`|解析并执行 `MainGameLoop`（会隐藏面板）|

**注意**

- 面板在应用前会把用户输入的所有空白字符（包括换行与空格）移除，然后交由 `Term.fromString` 解析。这意味着在编辑时你可以使用换行和空格提升可读性，但最终解析阶段空白会被删除并且不会影响解析（例如 `pair(a, b)` → `pair(a,b)`）。
    

---

### 3.2 “项”的通用语法规则（来自 `Term.fromString(str)`）

**分析来源**：下列语法规则直接由 `Term.fromString` 的实现导出（函数位于脚本开头）——这是解析有效项字符串的说明。

#### 语法概览（BNF 风格）

```
<term> ::= "var(" <ident> ")"            // 变量
         | <ident>                      // 常量
         | <ident> "(" <arg-list> ")"   // 复合项（构造子）
<arg-list> ::= "" | <term> ( "," <term> )*
<ident> ::= 由正则 \w+ 匹配（字母/数字/下划线）
```

**具体细则**

- 识别顺序：解析器先尝试 `var(...)`（变量），再尝试复合项 `<name>(...)`，最后尝试常量 `\w+`。不匹配时抛出 `Invalid term string` 错误。
    
- 标识符规则：`(\w+)`，即只允许英数字和下划线（`[A-Za-z0-9_]`）。不支持引号、破折号、空格、冒号等特殊字符作为标识符的一部分。
    
- `var(...)`：
    
    - 正则 `/^var\((\w+)\)$/i` —— `var` 关键字本身不区分大小写（`var`, `VAR`, `Var` 等都匹配），但变量名仍受 `\w+` 限制。
        
    - 变量在内部表示为 `Term.Var(name)`，并在匹配/替换中以变量名字符串（例如 `"X"`）作为替换表的键。
        
- 复合项 `name(arg1,arg2,...)`：
    
    - 名称必须匹配 `\w+`。
        
    - 支持**零参数**的复合项（写法 `name()`）；实现中对 `argsStr.trim() === ''` 会返回 `Term.Compound(name, [])`。
        
    - 参数分割：解析器通过简单的括号计数（balance）扫描并在最外层的逗号处分割参数 —— 允许嵌套括号，例如 `f(g(a,b),h(c))` 被正确解析。
        
- 常量：若字符串整体匹配 `/^\w+$/` 则视为常量 `Term.Const(name)`。
    
- 空白处理：
    
    - `Term.fromString` 在最开始会 `trim()` 输入（去除首尾空白）。
        
    - 调用方（UI 的构造/开发者面板）常常会去掉**所有空白字符**（`replace(/\s/g,'')`），但解析器本身仅去首尾空白。作为设计者，最好避免在标识符内使用空白（本来就不允许）。
        
- 错误处理：
    
    - 非法格式（如缺失括号、非法字符）会抛出异常，UI 会捕获并向用户显示 `无效的项格式`。
        

**示例（全部有效）**

- `apple` → 常量
    
- `var(X)` → 变量 X
    
- `pair(apple,green)` → 复合项，两个子项为常量
    
- `tuple()` → 零参数复合项
    
- `rev_pair(var(Y),var(X))` → 嵌套复合项含变量
    
- `puzzle_state(rewrite(...), data(...))` → 嵌套结构
    

---

### 3.3 核心构造函数详解

> 引言：在 TermCraft 中，大多数复合项名称仅用于组织数据（任意名称都可用于关卡状态），但有若干**特殊名称**（engine keywords）会被引擎识别并驱动游戏机制。理解这些关键字与普通构造子（data constructors）的区别，是设计复杂关卡的根本。

#### 3.3.1 机制驱动型构造函数（Engine Keywords）

下面列出并解释引擎在代码中直接识别、并对游戏行为产生影响的构造函数 / 常量。

---

**一、重写规则：`rewrite`**

- **结构**：`rewrite(LHS, RHS)`（必须二元）
    
- **作用**：定义可供玩家选择的**重写规则**。引擎将 `rewrite` 项视为规则并扫描它们来生成“可用动作”列表。
    
- **参数**：
    
    - `LHS`：模式项（可以包含变量 `var(...)`），用于匹配游戏中地面子项（或其等价类中的成员）。
        
    - `RHS`：替换项，在应用规则时会把 `LHS` 中绑定的变量代入到 `RHS` 中形成最终要插入的项。
        
- **机制（关键细节）**：
    
    - 在规则收集阶段，函数 `FindValidRewrites` 从项树中找到所有 `rewrite` 子项（通过 `FindAllSubtermsWithPositions`）。只接受 `arity === 2` 的 `rewrite` 项。
        
    - **安全检查**：若 RHS 引入了 LHS 中没有的变量（即 RHS 的自由变量不是 LHS 自由变量的子集），引擎会**忽略该规则**并在控制台警告（`忽略非法规则（R 引入未绑定变量）`）。
        
    - 匹配过程使用 `FindInferentialMatches`：它会先通过等价闭包（`equiv`）扩展候选匹配项，然后对每个等价类成员尝试**语法匹配**（`SyntacticMatch`），收集所有满足的替换（substitution）并去重（`UniqueSubstitutions`）。
        
    - 当玩家选择并实际执行某个重写动作，`ApplyAction` 会使用 `ApplySubstitution` 对 RHS 应用绑定，得到 `term_to_insert`，并以 `replaceSubtermAt` 将该处替换；引擎还会在应用时确保替换后不含变量（额外的安全检查）。
        
- **代价**：应用一条 `rewrite` 动作消耗 **1 燃料**（`F` 减 1）。
    
- **示例**：  
    `rewrite(pair(var(X), var(Y)), rev_pair(var(Y), var(X)))`  
    当 `pair(apple,green)` 在游戏中（或等价于它的项）被找到时，可以将该子项替换为 `rev_pair(green,apple)`。
    

---

**二、等价公理：`equiv`**

- **结构**：`equiv(TermA, TermB)`（应为二元）
    
- **作用**：声明两个**地面项（ground term）** 相互等价。`equiv` 本身不是玩家可以“应用”的动作，而是作为**背景信息**影响匹配与同余闭包构造。
    
- **参数**：
    
    - `TermA`, `TermB`：两端应为地面项（不能含变量）。若包含变量，`CalculateEquivalenceRelation` 会忽略并在控制台警告。
        
- **机制（关键细节）**：
    
    - `CalculateEquivalenceRelation`：
        
        1. 收集当前 `GameState.T` 中所有**地面子项**（`GetAllGroundSubtermsWithPositions`），把它们加入 `CongruenceClosure` 的初始项集中。
            
        2. 找到所有 `equiv` 子项（`FindAllSubtermsWithPositions(T, "equiv")`），并对每个地面 `equiv(a,b)` 把 `a` 和 `b` 加入相等对集合。
            
        3. 调用 `CongruenceClosure.buildClosure(eqPairs)`：先把显式等价对 union，然后通过构造子签名（`constructor|arity|[childRepStrings]`）对复合项做同余合并直到不再变化（fixpoint）。
            
    - **匹配影响**：当尝试把某处（一个地面子项）与 `rewrite` 的 LHS 匹配时，引擎**会把该处的等价类中的每个成员**都尝试与 LHS 做语法匹配 —— 因此 `equiv` 能发现因为同余而额外成立的匹配（比如 `equiv(apple,red)` 会让 `red` 与 `apple` 等价，从而匹配涉及 `apple` 的规则）。
        
- **示例**：  
    `equiv(apple, red)` —— 之后所有匹配引擎在匹配位置看到 `red` 时也会把 `apple` 视为可能匹配对象（反之亦然）。
    

---

#### 3.3.2 结构/数据型构造函数（Data Constructors）

**说明**：下面这些构造函数在示例关卡或界面中出现，但引擎不会把它们当成规则或影响匹配的关键字。它们只是**关卡作者用于构造状态的普通复合项名称**——你可以自由创建自己的名称来组织数据。

**示例（来源：`loadSampleLevel`）**

- `puzzle_state(...)`：通常作为关卡根节点包裹全部内容（规则、公理、数据）。只是容器而已。
    
- `data(...)`：示例中用来标记真正的**游戏数据**条目（例如 `data(pair(...))`、`data(placeholder)`）。引擎不对 `data` 名称加以特殊处理。
    
- `pair(...)`, `rev_pair(...)`：纯粹的数据/示例构造子，用来演示规则如何对数据起作用。它们没有内置语义，除非你写出 `rewrite` 指向它们。
    

**设计建议**：

- 可自由使用任意名称组织状态；关键是：**只有名称等于 `rewrite` 或 `equiv`（且格式满足）才会被当作规则/公理**。
    
- 为了可读性，建议把规则与数据用显式的不同构造子分开（例如 `rewrite(...)` 放在 `puzzle_state` 的前面，`data(...)` 放在后面），这样开发者面板与 UI 显示更清晰。
    

---

#### 3.3.3 特殊常量：`placeholder`

- **类型**：常量（写法 `placeholder` —— 必须是常量，不能是复合项或变量）
    
- **作用**：在 UI 中表现为黄色高亮的**可点击占位符**，玩家点击会弹出“构造新项”模态，用来把占位符替换为玩家构造的新地面项。
    
- **机制（由 `FindPlaceholderSites` 实现）**：
    
    - 仅识别**常量**并且 `term.getConstructor() === "placeholder"` 的项（函数 `FindPlaceholderSites` 有注释 `只接受常量形式的 placeholder`）。
        
    - 所有这样的占位符位置会在动作列表中被列为 `Construct` 动作，或在项树上直接响应点击以打开构造模态。
        
- **构造限制**：
    
    - 构造的新项必须是地面项（不能含变量），且 `Size(newItem)` ≤ 当前 `Budget`。
        
    - 构造过程还会在提交前去掉所有空白再解析；若解析失败会在模态中提示错误。
        

---

### 3.4 资源计算机制

**分析来源**：`Size(T)` 函数在代码中用于计算项的代价（构造时消耗 Budget），其实现非常直接：

```js
function Size(T) {
  let size = 1;
  if (T.isCompound()) size += T.children.reduce((acc, child) => acc + Size(child), 0);
  return size;
}
```

**解释（递归规则）**

- **基础：** 常量与变量的 `Size` = `1`。
    
- **复合项：** `Size(compound)` = `1 + Σ Size(child_i)`（对所有子项递归求和再加 1）。
    
    - 也可以理解为：每个节点（无论是否叶子）计 1，整棵子树的节点总数即为 `Size`。
        
- **含义：** `Size` 等价于项树中的节点数（每个 CONST / VAR / COMPOUND 节点计作 1）。
    

**计算示例**

- `Size(apple)` = 1
    
- `Size(var(X))` = 1
    
- `Size(pair(apple, green))` = 1 (pair 节点) + Size(apple) + Size(green) = 1 + 1 + 1 = **3**
    
- `Size(rev_pair(var(Y), var(X)))` = 1 + 1 + 1 = **3**
    
- `Size(puzzle_state(rewrite(...), data(...)))` = 1 + Size(rewrite(...)) + Size(data(...)) ...（按节点计数逐级展开）
    

**对关卡设计的影响**

- 构造更复杂的项（嵌套较深或含更多参数）会迅速增加预算代价。设计关卡时请用 `Size` 作为**显式的计量单位**：把预算设置为玩家需要但不会无限制滥用的数值。
    
- 在 UI/开发者面板中，构造模态会提示剩余预算（`construct-budget`），并在提交时校验 `Size`。
    

---

### 3.5 应用示例与注意事项

**如何构造一个可解关卡（最小步骤示例）**

1. 在 `puzzle_state(...)` 中放入所需的 `rewrite` 规则与 `equiv` 公理，并用 `data(...)` 填入初始数据与 `placeholder`：
    
    ```text
    puzzle_state(
      rewrite(pair(var(X), var(Y)), rev_pair(var(Y), var(X))),
      rewrite(red, blue),
      equiv(apple, red),
      data(pair(apple, green)),
      data(placeholder)
    )
    ```
    
2. 把 `Target Term` 设为包含 `data(rev_pair(green,blue))` 与 `data(done)` 等内容（示例里就是 `loadSampleLevel` 的做法）。
    
3. 为了让玩家可能达到目标，确保：
    
    - 有一条可用的 `rewrite` 能把 `pair(apple,green)` 转为 `rev_pair(green,apple)`；
        
    - 有一条 `rewrite(red, blue)` 能把 `apple`（通过 `equiv(apple, red)`）变成 `blue` —— 这里同余与 rewrite 的配合能产生 `rev_pair(green,blue)`；
        
    - 在必要位置定义 `placeholder` 让玩家构造额外常量（例如替换为 `done`）并留足预算。
        

**常见陷阱（必须避免）**

- **RHS 引入未绑定变量**：写 `rewrite(a, var(X))` 而 `a` 不包含 `var(X)` 会被引擎忽略。始终确保 `freeVars(RHS) ⊆ freeVars(LHS)`。
    
- **把 `equiv` 写成含变量的形式**：`equiv` 仅对地面项有效；含变量的 `equiv` 会被忽略（并出现控制台警告）。
    
- **占位符不是复合项/变量**：`FindPlaceholderSites` 只匹配常量等于 `"placeholder"`。若你写成 `placeholder()` 或 `var(placeholder)`，将不会被识别为可构造位点。
    
- **标识符中的非法字符**：`-`、空格、冒号等不被允许；用 `_` 代替空格（例如 `green_apple`）。
    
- **解析时的空白处理**：开发者面板会去掉所有空白再解析（包括换行），但 `Term.fromString` 本身只 `trim()` 首尾空白。因此在复杂字符串里避免在标识符间插入特殊字符。
    

**调试技巧**

- 在浏览器控制台中注意警告日志（脚本会在忽略规则或非法 `equiv` 时输出 `console.warn`）。
    
- 使用开发者面板先用小规模 `Fuel` / `Budget` 验证每一步是否按预想发生，再放开限制。
    
- `DisplayToPlayer` 会给可点击 placeholder 加上事件监听，通过点击直接打开构造模态，便于单步测试。
    

---

## 附：对重要函数/行为的逐行技术摘要

> 下面摘录并解释引擎中直接决定行为的关键实现细节（便于在设计复杂规则时精确预期引擎反应）。

### `Term.fromString(str)`（解析器）要点

- 先 `trim()` 输入。
    
- 如果匹配 `^var\((\w+)\)$`（不区分大小写），返回变量节点（`Term.Var(name)`）。
    
- 如果匹配 `^(\w+)\((.*)\)$`，则解析为复合项：
    
    - `name` = 第一组；`argsStr` = 第二组。
        
    - 若 `argsStr.trim() === ''` 则返回零参复合项。
        
    - 使用括号平衡计数在最外层按逗号分割参数；对每个子片段递归调用 `Term.fromString`。
        
- 如果匹配 `/^\w+$/`，返回常量 `Term.Const`。
    
- 否则抛出错误：`Invalid term string: "..."`。
    

### `FindValidRewrites(T, equiv_relation)` 要点

- 找出所有的 `rewrite` 子项（`FindAllSubtermsWithPositions(T,"rewrite")`）。
    
- 跳过不是二元的 `rewrite`。
    
- 计算 `fvL = FreeVars(LHS)` 与 `fvR = FreeVars(RHS)`，若 `fvR - fvL` 非空则忽略该规则（R 引入未绑定变量）。
    
- 对每个**地面**可能的应用位点（`GetAllGroundSubtermsWithPositions`）：
    
    - 使用 `FindInferentialMatches(l, site_term, equiv_relation)`：先从等价关系取等价类，再对每个成员做 `SyntacticMatch`（把变量名作为 substitution 的 key）。
        
    - 收集所有一致的 substitution（并去重），每个都生成一个 `Rewrite` 动作条目（含路径、规则路径、sigma、以及 snapshot 供 UI 显示）。
        
- 返回所有 `Rewrite` 动作（列表），供 UI 显示与玩家选择。
    

### `CalculateEquivalenceRelation(T)` 要点

- 收集所有地面子项（去重），把它们添加进 `CongruenceClosure.termMap`。
    
- 收集所有 `equiv` 子项（仅地面），把它们作为相等对传入 `CongruenceClosure.buildClosure`。
    
- `CongruenceClosure.buildClosure`：
    
    - 先对显式等价对做 union（并 `addTerm`）。
        
    - 然后在循环中对复合项按照签名 `constructor|arity|[childReps]` 建立映射：当两个复合项构造子名与子代表相同，执行 union，直到不再变化（fixpoint）。返回闭包对象（支持 `getClass` 查询等价类成员）。
        
- 结果用于扩展 `rewrite` 的匹配候选（等价类中的任何成员都可作为匹配对象进行语法匹配）。
    

### `FindPlaceholderSites(T)` 要点

- 只认 **常量** 并且 `getConstructor() === "placeholder"` 的节点（`isConstant()` + constructor 名称），返回这些节点的位置（path）。
    
- 因此 `placeholder()`（复合）不会被视作占位符；**必须写作常量 `placeholder`**。
    

### `Size(T)` 要点

- `Size` = 节点计数（叶子与内节点皆计 1）。
    
- 常量/变量：1；复合项 = 1 + Σ 子项 Size。
    

---

## 结语与设计建议

- **如果你是玩家**：将重心放在如何用有限燃料选择最有效的 `rewrite`，并用 `placeholder` 构造关键元素完成目标。多尝试点击动作卡并观察引擎展示的匹配替换与代价。
    
- **如果你是关卡设计师**：把握三要点即可制作可控、可解释的关卡：
    
    1. **规则合法性**：所有 `rewrite` 的 RHS 不引入 LHS 没有的变量；`equiv` 只用于地面项。
        
    2. **资源预算与 Size**：用 `Size` 控制构造成本，设置 Budget 限制玩家的创造力；把关键路径的节点数与 Budget 对齐。
        
    3. **清晰的数据组织**：把规则/公理与数据分离（例如 `puzzle_state(rewrite(...), data(...))`），便于 UI 显示与玩家理解。
