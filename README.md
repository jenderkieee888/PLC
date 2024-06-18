 ## 2023  － 2024   学年第 2 学期

 

 <img src="./img/logo.jpg" alt="img" style="zoom: 67%;" /> 

 

 

### 实 验 报 告

<img src="./img/logo2.jpg" alt="img" style="zoom:67%;" /> 

- 课  程  名   编程语言原理与编译     
- 课  程  号      C01102         
- 学 生 姓  名   熊博宁  肖涵瑜         
- 学 生 学 号      32002091         
- 专 业 班 级   计算2001  计算2003         
- 所 在 学 院    计算分院         
- 指 导 老 师     郭鸣        

 

 实验报告日期：  2024  年  6 月 20 日

---

## 简介
这是一个编译原理大作业，主要内容是使用F#语言完善microC，实现部分功能。

## 结构
- 前端：由`F#`语言编写而成  
  - `CLex.fsl`生成的`CLex.fs`词法分析器。
  - `CPar.fsy`生成的`CPar.fs`语法分析器。
  - `AbSyn.fs` 定义了抽象语法树
  - `Parse.fs` 是语法解析器
  - `Interp.fs`是解释器
  - `Comp.fs`将抽象语法树转化为中间表示
  
- 后端：由`Java`语言编写而成
  - `Machine.java`生成`Machine.class`虚拟机与`Machinetrace.class`堆栈追踪

- 测试集：测试程序放在`testing`文件夹内

- 库：`.net`8.0支持
## 用法

- ```
  dotnet restore  interpc.fsproj   # 可选
  dotnet clean  interpc.fsproj     # 可选
  dotnet build -v n interpc.fsproj
  ```


虚拟机功能：
- `java Machine` 运行中间表示
- `java Machinetrace` 追踪堆栈变化

例子：
```bash
java Machine ex11.out 8
java Machinetrace ex9.out 0
```

## 功能实现
- 带初始化的变量声明

- - 简介：原本的microC只有变量声明，我们改进了它使它具有变量定义，且在全局环境与local环境都具有变量定义的功能。

  - 对比 
    ```C
    // old
    int a;
    a = 3;
    int main(){
    
        print a;
    } 
    ```
    ```C
    // new 
    int a = 1;
    int b = 2 + 3;
    
    int main(){
        int c = 3;
        print a;
        print b;
        print c;
    }
    ```
  
  ### 1. 修改 `Absyn.fs`

  **原始代码：**

  ```fsharp
  and topdec = 
      ...
  ```
  
  **修改后代码：**
  
  ```fsharp
  and topdec = 
      ...
      | VardecInit of typ * string * expr  (* 带初始化的变量声明 *)
      ...
  ```
  
  **解释：** 在抽象语法树中添加一个新的节点类型 `VardecInit`，用于表示带初始化的变量定义。`VardecInit`包含变量名、变量类型和初始化表达式。
  
  ### 2. 修改 `CLex.fsl`
  
  无需修改
  
  ### 3. 修改 `CPar.fsy`
  
  **原始代码：**
  
  ```fsharp
  %token <string> IDENT
  %token IF ELSE
  ...
  %start stmt
  
  Topdec: 
      Vardec SEMI                         { Vardec (fst $1, snd $1) }
    | Fundec                              { $1 }
  ;
  
  Vardec:                                
      Type Vardesc                        { ((fst $2) $1, snd $2) }
  ;
  
  ...
  
  StmtOrDecSeq:
    ...
  ;
  
  
  ```
  
  **修改后代码：**
  
  ```fsharp
  %token <string> IDENT
  %token IF ELSE
  %token VAR EQ
  ...
  %start stmt
  
  Topdec: 
      Vardec SEMI                         { Vardec (fst $1, snd $1) }
    | VardecInit SEMI                     { VardecInit ($1) }
    | Fundec                              { $1 }
  ;
  
  Vardec:                                
      Type Vardesc                        { ((fst $2) $1, snd $2) }
  ;
  
  VardecInit:
      Type NAME ASSIGN Expr               { ($1, $2, $4) }
  ;
  
  StmtOrDecSeq:
    ...
    | VardecInit SEMI StmtOrDecSeq        { DecInit (fst $1, snd $1, thd $1) :: $3 }
  ;
  ```
  
  **解释：** 
  
  添加 `VardecInit` 规则，用于支持带初始化的变量声明。
  
  修改 `Topdec` 规则，添加对 `VardecInit` 的支持。
  
  修改 `StmtOrDecSeq` 规则，添加对 `VardecInit` 的支持。
  
  ### 4. 修改 `Parse.fs`
  
  **原始代码：**
  
  ```
  let parse (tokens : token list) : stmt =
      ...
  ```
  
  **修改后代码：**
  
  ```
  let parseVarDecl (tokens : token list) : stmt =
      match tokens with
      | VAR :: IDENT name :: COLON :: typ :: EQ :: expr :: SEMI :: rest ->
          VarDecl(name, typ, expr)
      | _ -> failwith "Invalid variable declaration"
  
  let parse (tokens : token list) : stmt =
      match tokens with
      ...
      | VAR :: _ -> parseVarDecl tokens
      ...
  ```
  
  **解释：** 在解析器中添加处理带初始化变量定义的逻辑。`parseVarDecl` 函数用于解析带初始化的变量定义，`parse` 函数中添加对 `VAR` 关键字的处理。
  
  ### 5. 修改 `Interp.fs`

  **原始代码：**
  
  ```
  let rec exec (stmt : stmt) (env : env) : env =
      match stmt with
      ...
  ```
  
  **修改后代码：**
  
  ```
  let rec exec (stmt : stmt) (env : env) : env =
      match stmt with
      ...
      | VarDecl(name, typ, expr) -> 
          let value = eval expr env
          addVar env name typ value
      ...
  ```
  
  **解释：** 在解释器中添加对带初始化变量定义的处理。`exec` 函数中添加对 `VarDecl` 节点的处理逻辑，通过调用 `eval` 函数计算初始化表达式的值，然后通过 `addVar` 函数将变量及其值添加到环境中。
  
  ### 6. 修改 `Comp.fs`

  **原始代码：**
  
  ```
  let rec compile (stmt : stmt) : instr list =
      match stmt with
      ...
  ```
  
  **修改后代码：**
  
  ```
  let rec compile (stmt : stmt) : instr list =
      match stmt with
      ...
      | VarDecl(name, typ, expr) ->
          let code = compileExpr expr
          code @ [DeclareVar(name, typ)]
      ...
  ```
  
  **解释：** 在编译器中添加对带初始化变量定义的处理。`compile` 函数中添加对 `VarDecl` 节点的处理逻辑，生成相应的声明变量指令和初始化表达式的字节码。
  
  ### 7. 修改 `Contcomp.fs`
  
  **原始代码：**
  
  ```
  let rec contCompile (stmt : stmt) : instr list =
      match stmt with
      ...
  ```
  
  **修改后代码：**

  ```
  let rec contCompile (stmt : stmt) : instr list =
      match stmt with
      ...
      | VarDecl(name, typ, expr) ->
          let code = contCompileExpr expr
          code @ [DeclareVar(name, typ)]
      ...
  ```
  
  **解释：** 在优化编译器中添加对带初始化变量定义的处理。`contCompile` 函数中添加对 `VarDecl` 节点的处理逻辑，生成相应的声明变量指令和初始化表达式的字节码。
  
  ### 测试样例和测试命令
  
  ### 测试样例 1
  
  #### 代码：`example/test1.c`
  
  ```
  int a = 1;
  
  void main() {
      print(a);
  }
  ```
  
  #### 解释器测试命令：
  
  ```
  dotnet run --project interpc.fsproj example/test1.c
  ```
  
  #### 编译器测试命令：
  
  ```
  dotnet run --project microc.fsproj example/test1.c
  dotnet run --project microcc.fsproj example/test1.c
  ./bin/Debug/net8.0/machine.exe example/test1.out
  ```
  
  ### 测试样例 2
  
  #### 代码：`example/test2.c`
  
  ```
  int b = 2 + 3;
  
  void main() {
      print(b);
  }
  ```
  
  #### 解释器测试命令：
  
  ```
  dotnet run --project interpc.fsproj example/test2.c
  ```
  
  #### 编译器测试命令：
  
  ```
  dotnet run --project microc.fsproj example/test2.c
  dotnet run --project microcc.fsproj example/test2.c
  ./bin/Debug/net8.0/machine.exe example/test2.out
  ```

---

- float 类型
  - 简介：浮点型类型，我们将原本的小数转化为二进制表示，再将这二进制转化为整数放入堆栈中，在虚拟机中再转化为小数。
  - 例子：
    - 例一：   
        ```C
        int main(){
            float a = 1.0;
            print a;
        }
        ```
    - 运行栈追踪：  
        ![](img/ex(float)1.png)

    - 例二：
        ```C
        int main(){
            float a = 1.0;
            int b = 2;
            print a+b;
        }
        ```
    - 运行栈追踪：  
        ![](img/ex(float)2.png)

---

- char 类型
  - 简介：原本电脑
  - 例子：

    ```C
    int main ()
    {
        char c = 'c';
        print c;
    
    }
    ```
  - 运行栈追踪：  
    ![](img/ex(chars).png)


---
- 自增操作
    - 简介:包含i++ ++i 操作
    - 例子：
        ```C
        int main(){
            int n;
            int a;
            n = 2;
            a = ++n;
            a = n++;
        }
        ```
    - 运行栈追踪：  
        ![](img/ex(selfplus).png)
---
- FOR循环
    - 简介：增加了for循环
    
    - 过程：
    
        ### 1. Absyn.fs
    
        #### 初始
    
        ```fsharp
        type stmt =
            | ...
            // Other cases
        ```
    
        #### 修改
    
        ```fsharp
        type stmt =
            ...
            | For of expr * expr * expr * stmt
        ```
    
        把这个更改添加到 `Absyn.fs` 文件中。
    
        接下来，更新 `CLex.fsl` 文件以识别 `for` 关键字。
    
        ### 2. CLex.fsl
    
        #### 初始
    
        ```fsharp
        let keyword s =   
            ...
            // Other rules
        ```
    
        #### 修改
    
        ```fsharp
        let keyword s =  
            ...
            | "for"     -> FOR
        ```
    
        把这个更改添加到 `CLex.fsl` 文件中。
    
        接下来，更新 `CPar.fsy` 文件以包含 `for` 循环的解析规则。
    
        ### CPar.fsy
    
        #### 初始
    
        ```fsharp
        %token <string> IDENT
        %token <int> INT
        %start start
        %type <Absyn.program> start
        
        %%
        
        stmt:
            ...
            // Other rules
        ```
    
        #### 修改
    
        ```fsharp
        %token <string> IDENT
        %token <int> INT
        %token FOR
        %start start
        %type <Absyn.program> start
        
        %%
        
        stmt:
            ...
            | FOR LPAR Expr SEMI Expr SEMI Expr RPAR Stmt { For($3, $5, $7, $9) }
        ```
    
        我将把这个更改添加到 `CPar.fsy` 文件中。
    
        接下来，让我们更新 `Parse.fs` 文件以确保解析器可以处理新的 `For` 节点。
    
        ### Parse.fs
    
        #### 初始
    
        ```fsharp
        module Parse
        
        open System
        open FSharp.Text.Lexing
        open Absyn
        open CLex
        open CPar
        
        let fromFile (filename : string) =
            let lexbuf = LexBuffer<char>.FromFile filename
            let res = CPar.start CLex.tokens lexbuf
            res
        ```
    
        `Parse.fs` 中不需要进行任何更改，因为它已经正确处理了抽象语法树定义。
    
        ### 5. Interp.fs
    
        添加`for`循环的解释逻辑。
    
        #### Interp.fs
    
        ```fsharp
        module Interp
        
        open Absyn
        
        let rec exec stmt (locEnv: locEnv) (gloEnv: gloEnv) (store: store) : store =
            ...
            | For (init, cond, step, body) ->
                // 执行 For 循环初始化
                let (_, store1) = eval init locEnv gloEnv store
            
                // 定义 For 循环辅助函数 loop
                let rec loop store1 =
                    let (v, store2) = eval cond locEnv gloEnv store1
                    if v <> 0 then
                        let store3 = exec body locEnv gloEnv store2
                        let (_, store4) = eval step locEnv gloEnv store3
                        loop store4
                    else
                        store2 // 退出循环返回 store2
                loop store1
        ```
    
        我将把这个更改添加到 `Interp.fs` 文件中。
    
        接下来，更新 `Comp.fs` 文件以包含 `for` 循环的编译逻辑。
        
        ### 6. Comp.fs
        
        #### 修改前：
        
        ```fsharp
        module Comp
        
        open Absyn
        open StackMachine
        
        let rec cStmt stmt (varEnv: VarEnv) (funEnv: FunEnv) : instr list =
            ...
            // Other cases
        ```
        
        #### 修改后：
        
        ```fsharp
        module Comp
        
        open Absyn
        open StackMachine
        
        let rec cStmt stmt (varEnv: VarEnv) (funEnv: FunEnv) : instr list =
            ...
            | For (init, cond, incr, body) ->
                let initCode = cExpr init varEnv funEnv @ [ INCSP -1 ]
                let labbegin = newLabel ()
                let labtest = newLabel ()
                let bodyCode = cStmt body varEnv funEnv
                let incrCode = cExpr incr varEnv funEnv @ [ INCSP -1 ]
        
                initCode @ [ GOTO labtest; Label labbegin ]
                @ bodyCode
                @ incrCode
                @ [ Label labtest ]
                @ cExpr cond varEnv funEnv @ [ IFNZRO labbegin ]
        ```
    
        我将把这个更改添加到 `Comp.fs` 文件中。
        
        最后，让我们更新 `Contcomp.fs` 文件以包含 `for` 循环的连续编译逻辑。
        

- 测试：

    ### 示例程序：fortest.c

    ```
    void main(int n) {
        int i;
        for (i = 0; i < n; i = i + 1) {
            print i;
        }
    }
    ```

    ### 测试步骤：

    1. **编译解释器**：

       ```bash
       dotnet restore interpc.fsproj
       dotnet clean interpc.fsproj
       dotnet build -v n interpc.fsproj
       dotnet run --project interpc.fsproj example/fortest.c 8
       ```

    2. **编译编译器**：

       ```bash
       dotnet restore microc.fsproj
       dotnet clean microc.fsproj
       dotnet build microc.fsproj
       dotnet run --project microc.fsproj example/fortest.c 8
       ```

    3. **运行编译后的代码**：

       ```bash
       ./bin/Debug/net8.0/machine.exe ./example/fortest.out 8
       ```

- 结果：  

    ![for](D:\plc\for.png)

    

---
- 三目运算符
    - 简介：三目运算符 a>b?a:b
    - 用例：
        ```C
        int main()
        {
            int a=0;
            int b=7;
            int c = a>b?a:b;
        }
        ```
    - 运行栈追踪：  
        ![](img/ex(ternary).png)
---
- do - while
    - 简介：在判断前先运行body中的操作。
    - 例子：
        ```C
        int main()
        {
            int n=2;
            do{
                n++;
            }while(n<0);
        }
        ```

  - 运行栈追踪：
    - n++被执行
    - n的终值为3 处于栈中2的位置
    - 堆栈图：
    ![](img/ex(dowhile).png)
---
- 类似C的switch-case
    - 当没有break时，匹配到一个case后，会往下执行所以case的body
    - 若当前没有匹配的case时，不会执行body，会一直往下找匹配的case
    - 之前的实现是递归匹配每个case，当前类似C语言的switch-case实现上在label的设立更为复杂一些。
    - 例子：
        ```C
        int main(){
            int i=0;
            int n=1;
            switch(n){
                case 1:i=n+n;
                case 5:i=i+n*n;
            }
        }
        ```

  - 运行栈追踪：
    - n的值与case1 匹配，没有break， i=n+n与case 5 中的i+n*n都被执行
    - i的结果为（1+1）+1*1 = 3
    - 栈中3的位置为i，4的位置为n
    - 堆栈图：  
    ![](img/ex(switch).png)

---

- break功能
    - 在for while switch 中，都加入break功能
    - 维护Label表来实现
    - 例子：与没有break的switch进行对比：
        ```C
        int main(){
            int i=0;
            int n=1;
            switch(n){
                case 1:{i=n+n;break;}
                case 5:i=i+n*n;
            }
        }
        ```
  - 运行栈追踪
    - n的值与case1 匹配，执行i=n+n，遇到break结束。
    - i的结果为（1+1）=2
    - 栈中3的位置为i，4的位置为n
    - 堆栈图：  
        ![](img/ex(break).png)

---
- continue 功能
    - 在for while 中加入continue功能
    - 例子：
        ```C
        int main()
        {
            int i ;
            int n = 0;
            for(i=0;i<5;i++)
            {
                if(i<2)
                    continue;
                if(i>3)
                    break;
                n=n+i;
            }
        }
        ```
  - 运行栈追踪：
    - i=0 1 的时候continue i>3 的时候break
    - n = 2 + 3 结果为5
    - 栈中3的位置为i， 4的位置为n
    - 堆栈图：  
    ![](img/ex(continue).png)

---

- 结构体功能：
  - 简介：加入了C中的结构体功能，可以嵌套数组
  - 
  - 例子：
    ```C
        struct student{
            int number;
            int number2;
            char name[5];
            float id;
        };
        int main(){
            struct student hello;
            hello.number = 10;
            hello.id = 234;
            hello.name[4] = 'c';
            hello.name[0] = 'a';
            print hello.number;
            print hello.name[4];
        }
    ```

    - 运行栈追踪
    - ![](img/ex(struct).png)
---

- JVM
  - 简介：
    - 将之前的虚拟机重新写了一边，原先的虚拟机只能针对int类型进行存储和操作。我具体定义了一些类型，使虚拟机具有较强的拓展性。
  - 类型继承关系图：
    - ![](img/JVM.png)
  - 指令集添加：
    - CSTF：
      - 简介：const float
      - 功能：在堆栈中加入一个float
    - CSTC： 
      - 简介：const char
      - 功能：在堆栈中加入一个char
  - 运行时异常：
    - OperatorError:
      - 简介：在计算时发生的异常，例如除数为0时，出现异常。
    - ImcompatibleTypeError:
      - 简介：类型不匹配的时候进行时出现的异常，例如'a' + 1.0 抛出异常。

---

- try-catch：
  - 简介：
    - 除0异常捕获：
      - 虚拟机运行时的除0异常
      - 出现显式的除0时THROW 异常
    - 寄存器添加：
      - hr： 保存当前异常在栈中的地址，用于追踪需要处理的异常
    - 指令添加：
      - PUSHHDLR,保存catch中的异常种类，需要跳转的位置以及hr入栈
      - POPHDLR ，与PUSHHDLR对应
      - THROW   ，用于丢出异常，从hr开始找匹配
    - 例子：
  - 目前做了有关除0异常的内容，由于没有异常类，暂且通过字符串的方式作为异常种类，将异常编号匹配。解决了运行时try-catch变量环境的问题，解决了异常处理时栈环境变化带来的空间影响。能够正常的匹配到异常。
    ```C
        int main()
        {
            try{
                int a=0;
                int n=5;
                n=n/a;
            }
            catch("ArithmeticalExcption")
            {
                n=0;
                print n;
            }   
        }
    ```
    - 运行时堆栈：  

- ![](img/ex(exception).png)
      


## 心得体会
- 熊博宁：  
  
- 肖涵瑜：  
  

## 技术评价

| 功能 | 对应文件 | 优  | 良  | 中  |
| ---- | -------- | --- | --- | --- |
|变量声明定义|test_init.c||||
|识别标识符_|test_var_underline.c|√|||
|注释（**）|test_comment.c|√|||
|自增、自减| test_incr.c          |√|||
|多进制|test_index.c|√|||
|位运算|test_bit_cal.c|√|||
|for循环|test_for.c|√|||
|三目运算符|test_ternary.c|√|||
|do-while|test_do_while.c|√|||
|while|test_while.c|√|||
|range循环|ex(range).c||||
|break|ex(break).c||||
|continue|ex(continue).c||||
|switch-case|ex(switch).c||||
|float 类型|ex(float).c||||
|char 类型|ex(chars).c||||
|struct结构体|ex(struct).c||||
|try-catch|ex(exception).c||||
|虚拟机类型支持|Machine.java||||
|虚拟机异常|exception||||



## 小组分工

- 熊博宁
  - 学号：31601147
  - 班级：计算机1603
    - 工作内容
      - 文档编写
      - 测试程序
      - 主要负责虚拟机和中间代码生成
      - 结构体
- 肖涵瑜
  - 学号：31601149
  - 班级：计算机1603
    - 工作内容
      - 文档编写
      - 测试程序
      - 语法分析
      - 词法分析
      - 栈、堆设计
      - try-catch

- 权重分配表：  

| 胡煜 | 江瑜 |
| ---- | ---- |
| 0.95 | 0.95 |

## 评分表


| 评分标准 | 总分 | 说明 | 自评计分 | 教师计分 |
| :---: | :---: | :---: | ----- | ----- |
| **代码质量** | 30分 | 代码结构清晰，易于理解；代码注释充足，有助于理解代码功能；代码无冗余，遵循DRY原则 | 27 |  |
| **功能实现** | 40分 | 所有要求的功能都已实现并正确运行；程序无明显错误或bug | 32 |  |
| **创新性** | 10分 | 作业展示了独特的创新思维；作业中实现的功能超出了基本要求 | 10 |  |
| **用户体验** | 10分 | 程序界面友好，易于操作；程序反馈清晰，易于理解 | 10 |  |
| **文档和报告** | 10分 | 提交了完整的文档，包括设计思路、代码解释和用户指南；提交了清晰的项目报告，包括项目概述、实现过程和遇到的问题及解决方案 | 10 |  |
| 总分 | 100分 |  | 89 |  |

 


## 指导教师评语

 

  **优：** "此份编译原理大作业表现出色，展示了深入的课程内容理解和应用。编译器设计结构清晰，模块划分合理，显示出良好的软件工程素养。词法分析器和语法分析器都能准确地识别和处理各种输入，这是非常值得赞扬的。代码优化部分也做得很好，实现了一些高级的优化技术。总的来说，此份作业完成得非常好，期待看到在编译器设计方面的进一步成果。" 

**良：** "此份编译原理大作业完成得很好，展示了对课程内容的理解和应用。编译器设计结构清晰，模块划分合理。词法分析器和语法分析器都能处理大部分的输入，这是值得赞扬的。然而，代码优化部分还有一些提升的空间。建议在未来的学习中，可以更深入地研究这些高级优化技术。" 

**中：** "此份编译原理大作业完成得一般，展示了对课程内容的基本理解。编译器设计有一定的结构，但模块划分还有待改进。词法分析器和语法分析器能处理一部分的输入，但还有一些错误需要修正。代码优化部分还有很大的提升空间。建议在未来的学习中，加强对编译原理的理解和应用。" 

**及格：** "此份编译原理大作业完成得基本合格，但还有很多需要改进的地方。编译器设计需要更清晰的结构和更合理的模块划分。词法分析器和语法分析器还有很多错误需要修正。代码优化部分几乎没有实现。建议在未来的学习中，加强对编译原理的理解和应用，提高编程技能。"

  

  实验报告评分（百分制）：   分

 

  指导教师签名：<img src="./img/签名.jpg" style="zoom:15%;" />

  日     期：2024年6  月 20 日
