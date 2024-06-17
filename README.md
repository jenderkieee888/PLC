# PLC Project

This expression was expected to have type
    'int env'    
but here has type
    '(string * address) list * address'F# Compiler1

### interpreter_test.sh
``` sh
#!/bin/zsh

dotnet restore  interpc.fsproj
dotnet clean  interpc.fsproj
dotnet build -v n interpc.fsproj

dotnet run --project interpc.fsproj example/vardec_test.c
```

### compiler_test.sh
``` sh
#!/bin/zsh

dotnet restore microc.fsproj
dotnet clean microc.fsproj
dotnet build microc.fsproj

dotnet run --project microc.fsproj example/for_test.c

./bin/Debug/net8.0/machine ./example/for_test.out 20

```
### 修改文件权限
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

| Vardec SEMI StmtOrDecSeq { Dec (fst $1, snd $1) :: $3 }

### CPAR自增
    | Incr acc ->
        let (loc, store1) = access acc locEnv gloEnv store
        let value = getSto store1 loc
        let newValue = value + 1
        (newValue, setSto store1 loc newValue)
    | Decr acc ->
        let (loc, store1) = access acc locEnv gloEnv store
        let value = getSto store1 loc
        let newValue = value - 1
        (newValue, setSto store1 loc newValue)

### CLex.fsl
``` fs
{
 (* File MicroC/CLex.lex
    Lexer specification for micro-C, a small imperative language
  *)

// module CLex = 
  
open FSharp.Text.Lexing
open CPar  // Token 的定义位于语法分析模块中

// 辅助函数，从lexbuf 得到当前的 词元字符串，lexeme 
let lexemeAsString lexbuf = 
    LexBuffer<char>.LexemeString lexbuf

(* Scan keywords as identifiers and use this function to distinguish them. *)
(* If the set of keywords is large, use a hashtable instead.               *)
// keyword 处理关键字与标识符的辅助函数
let keyword s =   
    match s with
    | "char"    -> CHAR    // 返回对应的Token ,Token 定义于 CPar.fsl 文件中
    | "else"    -> ELSE
    | "false"   -> CSTBOOL 0
    | "if"      -> IF
    | "int"     -> INT
    | "null"    -> NULL
    | "print"   -> PRINT
    | "println" -> PRINTLN
    | "return"  -> RETURN
    | "true"    -> CSTBOOL 1
    | "void"    -> VOID 
    | "while"   -> WHILE         
    | "for"   -> FOR
    | _         -> NAME s   // 缺省情况，是标识符的名字，如 函数名，变量名等
                            // 当所有的关键字没有匹配成功时，则当成标识符
// 字符串转义符处理函数
let cEscape s = 
    match s with 
    | "\\\\" -> '\\'
    | "\\\"" -> '\"'
    | "\\a"  -> '\007'
    | "\\b"  -> '\008'
    | "\\t"  -> '\t'
    | "\\n"  -> '\n'
    | "\\v"  -> '\011'
    | "\\f"  -> '\012'
    | "\\r"  -> '\r'
    | _      -> failwith "Lexer error: impossible C escape"
}
                        
rule Token = parse          // 每个 规则rule 会生成为一个函数，函数名是规则名 Token
  | [' ' '\t' '\r'] { Token lexbuf } 
                             // 变量是lexbuf 函数Token的默认参数，由外部程序构造
                             // 每条规则匹配成功后 lexbuf 内容会自动更新
                             // 缓冲区的读写指针后移，将匹配到的内容从lexbuf读出

  | '\n'            { lexbuf.EndPos <- lexbuf.EndPos.NextLine; Token lexbuf } // 换行处理
                                                                             // EndPos 是内置类型 Position的实例，表示当前行的结束位置
  | ['0'-'9']+      { CSTINT (System.Int32.Parse (lexemeAsString lexbuf)) }
                            //  System.Int32.Parse 将 字符串转换为 int 类型
                            // 在 CPar 中  词元CSTINT 的语义值的类型是int  %token <int> CSTINT
                            //  CSTINT 需要一个整数作为参数

  | ['a'-'z''A'-'Z']['a'-'z''A'-'Z''0'-'9']*
                    { keyword (lexemeAsString lexbuf) } //标识符处理
                                                       //先查是否是关键字，是则返回对应关键字的Token
  | '+'             { PLUS }  // 返回Token PLUS 给调用词法分析器的程序，通常是Parser
                              // 此时lexbuf是 匹配 '+'完成后，后续的内容 
  | '-'             { MINUS } 
  | "++"            { INCR }
  | "--"            { DECR }
  | '*'             { TIMES } 
  | '/'             { DIV } 
  | '%'             { MOD }                     
  | '='             { ASSIGN } 
  | "=="            { EQ } 
  | "!="            { NE }
  | '>'             { GT }
  | '<'             { LT }
  | ">="            { GE }
  | "<="            { LE }
  | "||"            { SEQOR }                     
  | "&&"            { SEQAND }                     
  | "&"             { AMP }                     
  | "!"             { NOT }                     
  | '('             { LPAR }
  | ')'             { RPAR }
  | '{'             { LBRACE }
  | '}'             { RBRACE }
  | '['             { LBRACK }
  | ']'             { RBRACK }
  | ';'             { SEMI }
  | ','             { COMMA }
  | "//"            { EndLineComment lexbuf; Token lexbuf }  //单行注释处理规则，调用响应处理函数
                                                             //参数是 lexbuf
                                                             // 处理完后 lexbuf 内容已经更新，注释部分过滤
                                                             //调用 Token 规则函数继续注释部分后面的处理
  | "/*"            { Comment lexbuf; Token lexbuf }   // 多行注释，调用 Comment规则
  | '"'             { CSTSTRING (String [] lexbuf) }   // 调用字符串处理规则
  | eof             { EOF }
  | _               { failwith "Lexer error: illegal symbol" }
  


and Comment = parse
   | "/*"           { Comment lexbuf; Comment lexbuf }  // 注释的嵌套处理
   | "*/"           { () }                           // 注释处理结束
   | '\n'           { lexbuf.EndPos <- lexbuf.EndPos.NextLine; Comment lexbuf } //注释跨行处理
   | (eof | '\026') { failwith "Lexer error: unterminated comment" }  // 多行注释未封闭
   | _              { Comment lexbuf }  // 其他任意情况都继续处理后续字符

and EndLineComment = parse
   | '\n'           { lexbuf.EndPos <- lexbuf.EndPos.NextLine }  //更新行尾位置，返回
   | (eof | '\026') { () }   // 文件结束，26 是 CTRL+Z的ASCII码，也是结束符 , () 退出返回
   | _              { EndLineComment lexbuf }  // 继续读lexbuf 中下个字符

and String chars = parse 
  | '"'
      { Microsoft.FSharp.Core.String.concat "" (List.map string (List.rev chars)) } 
                                                        // 字符串结束，通过字符数组chars构造字符串
                                                        // 由于构造的时候是列表 cons ::操作
                                                        // 这里需要用List.rev 翻转字符数组
  | '\\' ['\\' '"' 'a' 'b' 't' 'n' 'v' 'f' 'r']   //字符串 "\a" 读入后词法分析器 看到的是 "\\a"
      { String (cEscape (lexemeAsString lexbuf) :: chars) lexbuf }
  | "''" 
      { String ('\'' :: chars) lexbuf }
  | '\\'
      { failwith "Lexer error: illegal escape sequence" }
  | (eof | '\026')       { failwith "Lexer error: unterminated string" } 
                                                   // 字符串中出现文件结束

  | ['\n' '\r']  { failwith "Lexer error: newline in string" }  //字符串中出现回车

  | ['\000'-'\031' '\127' '\255']    
      { failwith "Lexer error: invalid character in string" }  // 字符串中出现 ASCII 控制字符
  | _
      { String (char (lexbuf.LexemeChar 0) :: chars) lexbuf } // 将读到的第1个字符加到临时的chars数组

```

### comp自增
    | Incr acc ->
        cAccess acc varEnv funEnv @ [ DUP; LDI; CSTI 1; ADD; STI ]
    | Decr acc ->
        cAccess acc varEnv funEnv @ [ DUP; LDI; CSTI 1; SUB; STI ]

### comp三目
    | Ternary (cond, e1, e2) ->
        let labtrue = newLabel ()
        let labend = newLabel ()
        cExpr cond varEnv funEnv
        @ [ IFNZRO labtrue ]
          @ cExpr e2 varEnv funEnv
            @ [ GOTO labend
                Label labtrue ]
          @ cExpr e1 varEnv funEnv
            @ [ Label labend ]

### test_do_while.c
``` c
void main()
{
    int i;

    i=10;

    do
    {
        print i;
        i--;
    } while (i>=0);
    
    println;

}
```

