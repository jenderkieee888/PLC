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
