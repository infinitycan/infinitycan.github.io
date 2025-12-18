---
layout: post
title: "Interpreter解析器模式"
subtitle: ""
date: 2025-12-18
author: "Can"
header-img: "img/background/moon.jpg"
tags: ["Design Pattern"]
---

## Motivation

- 在软件构建过程中，如果某一特定领域的问题比较复杂，类似的结构不断重复出现，如果使用普通的编程方式来实现将面临非常频繁的变化。
- 在这种情况下，将特定领域的问题表达为某种语法规则下的句子，然后构建一个解释器来解释这样的句子，从而达到解决问题的目的。

## Defination

给定一个语言，定义它的文法的一种表示，并定义一种解释器，这个解释器使用该表示来解释语言中的句子。

## Structure

![interpreter](/img/design-pattern/interpreter.png)

## Example

以一个简单的算术表达式解析器（支持整数加减法）为例

### Abstract Expression

```cpp
class Expression {
public:
    virtual ~Expression() = default;
    // 解释逻辑，通常会传入一个 Context 来保存全局状态
    virtual int interpret() = 0; 
};
```

### Terminal Expression

```cpp
// 实现与文法中的终结符相关的解释操作。在这里是数字。
class NumberExpression : public Expression {
private:
    int value;
public:
    NumberExpression(int v) : value(v) {}
    int interpret() override {
        return value;
    }
};
```

### Non-terminal Expression

```cpp
// 为文法中的规则实现解释操作。在这里是加法和减法。
// 加法解释器
class AddExpression : public Expression {
private:
    Expression* left;
    Expression* right;
public:
    AddExpression(Expression* l, Expression* r) : left(l), right(r) {}
    int interpret() override {
        return left->interpret() + right->interpret();
    }
};

// 减法解释器
class SubtractExpression : public Expression {
private:
    Expression* left;
    Expression* right;
public:
    SubtractExpression(Expression* l, Expression* r) : left(l), right(r) {}
    int interpret() override {
        return left->interpret() - right->interpret();
    }
};
```

### Client

```cpp
// 构建抽象语法树（AST）并执行解释。
int main() {
    // 表示表达式: 10 + 5 - 2
    
    // 1. 构建语法树
    // (10 + 5)
    Expression* e1 = new NumberExpression(10);
    Expression* e2 = new NumberExpression(5);
    Expression* add = new AddExpression(e1, e2);
    
    // ((10 + 5) - 2)
    Expression* e3 = new NumberExpression(2);
    Expression* subtract = new SubtractExpression(add, e3);
    
    // 2. 解释执行
    int result = subtract->interpret();
    
    std::cout << "Result: " << result << std::endl; // 输出 13
    
    // 3. 释放内存
    delete e1; delete e2; delete e3;
    delete add; delete subtract;
    
    return 0;
}
```

### Explanation

- **抽象语法树 (AST)：**解释器模式的核心通常在于客户端如何构建这棵树。在实际应用中，通常会配合 `词法分析器` 和 `语法分析器` 来自动生成这棵树，而**不是手动 new**。
- **Context（环境类）：**在上面的简单例子中没有体现，但在复杂的解释器中，interpret(Context* ctx) 会携带一个环境对象，用于存储变量名与数值的映射（如 x + y 中的 x 和 y）。

## Conclusion

- Interpreter模式的应用场合是Interpreter模式应用中的难点，只有满足“业务规则频繁变化，且类似的结构不断重复出现，并且容易抽象为语法规则的问题”才适合使用Interpreter模式。  
- 使用Interpreter模式来表示文法规则，从而可以使用面向对象技巧来方便地“扩展”文法。  
- Interpreter模式比较适合简单的文法表示，对于复杂的文法表示，Interpreter模式会产生比较大的类层次结构，需要求助于语法分析生成器这样的标准工具。
