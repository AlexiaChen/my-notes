
### OLAP技术设计

```txt
现在做olap已经有点搭积木的意思了。parser有antlr4, ogical plan有substrait, plan optimization有orca, codegen有gandiva, simd有arrow/orc/parquet, memory pool有jemalloc/tcmalloc。只有execution还不知道是否有相关组件？
```

