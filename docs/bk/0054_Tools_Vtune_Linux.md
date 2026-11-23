-----

| Title     | Tools Vtune Linux                                 |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-20T18:09:44Z`                            |
| Updated @ | `2024-04-20T18:09:44Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/54) |

-----

# Vtune Linux

## 推荐编译选项

  - `-g`: 生成符号信息，关联代码和更好的查看调用堆栈
  - `Release Build` or `-O2`: 最大编译优化

## Debug Information

  - 如果没有 `Debug Information` , 则会在调用堆栈中产生 `func@address` 的调用名
