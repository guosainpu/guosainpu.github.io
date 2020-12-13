---
title: Clang
tags: 计算机基础 工具 汇编语言
---

### 简介

1. Clang是C、C++、Objective-C的编译器前端，采用LLVM作为其后端，它的目标是代替GCC。
2. Clang包括Clang前端和Clang静态分析器。
3. 编译器前端的目标是输出：抽象语法树
4. Clang本身性能优异，其生成的AST所耗用掉的内存仅仅是GCC的20%左右。测试证明Clang编译Objective-C代码时速度为GCC的3倍，还能针对用户发生的编译错误准确地给出建议。