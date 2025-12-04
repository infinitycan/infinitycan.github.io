---
layout: post
title: "Abstract Factory抽象工厂模式"
subtitle: "如何创建一系列相互依赖的对象"
date: 2025-11-12
author: "Can"
header-img: "img/nature-4.jpg"
tags: ["Design Pattern", "C++"]
---

## 模式定义

提供一个接口，让该接口负责创建一系列“相关或者相互依赖的对象”（系列），无需指定它们具体的类。

## 模式结构

![Abstract Factory模式结构](/img/in-post/abstract-factory.png)

- **AbstractFactory**：抽象工厂类，声明了一个或多个创建抽象产品的方法。
- **ConcreteFactory**：具体工厂类，实现了抽象工厂类中声明的创建抽象产品的方法，生成一组具体产品。
- **AbstractProduct**：抽象产品类，为每种产品声明一个接口，描述产品的行为。
- **Product**：具体产品类，实现了抽象产品类中声明的接口，定义了具体产品的行为。

## 示例

### 抽象产品(Abstract Products)

```cpp
// 抽象产品 A：键盘
class AbstractKeyboard {
public:
    virtual ~AbstractKeyboard() {}
    // 定义键盘的抽象操作
    virtual void type() const = 0; 
};

// 抽象产品 B：鼠标
class AbstractMouse {
public:
    virtual ~AbstractMouse() {}
    // 定义鼠标的抽象操作
    virtual void click() const = 0;
    // 定义一个可以与另一个产品家族协同的操作
    virtual void workWith(const AbstractKeyboard* keyboard) const = 0; 
};
```

### 具体产品(Concrete Products)

```cpp
// 戴尔 (Dell) 家族产品 A
class DellKeyboard : public AbstractKeyboard {
public:
    void type() const override {
        std::cout << "Dell Keyboard: Typing with a focus on durability." << std::endl;
    }
};

// 戴尔 (Dell) 家族产品 B
class DellMouse : public AbstractMouse {
public:
    void click() const override {
        std::cout << "Dell Mouse: Clicking precisely." << std::endl;
    }
    void workWith(const AbstractKeyboard* keyboard) const override {
        std::cout << "Dell Mouse: Ensuring full compatibility with Dell Keyboard." << std::endl;
    }
};

// 苹果 (Apple) 家族产品 A
class AppleKeyboard : public AbstractKeyboard {
public:
    void type() const override {
        std::cout << "Apple Keyboard: Typing with a sleek, low-profile design." << std::endl;
    }
};

// 苹果 (Apple) 家族产品 B
class AppleMouse : public AbstractMouse {
public:
    void click() const override {
        std::cout << "Apple Mouse: Clicking silently." << std::endl;
    }
    void workWith(const AbstractKeyboard* keyboard) const override {
        // 这里的协同操作可以体现家族特色
        std::cout << "Apple Mouse: Optimized for macOS gesture control." << std::endl;
    }
};
```

### 抽象工厂(Abstract Factory)

```cpp
class AbstractFactory {
public:
    virtual ~AbstractFactory() {}
    // 创建家族中的第一个产品（键盘）
    virtual AbstractKeyboard* createKeyboard() const = 0;
    // 创建家族中的第二个产品（鼠标）
    virtual AbstractMouse* createMouse() const = 0;
};
```

### 具体工厂(Concrete Factory)

```cpp
// 戴尔 (Dell) 家族工厂
class DellFactory : public AbstractFactory {
public:
    AbstractKeyboard* createKeyboard() const override {
        return new DellKeyboard();
    }
    AbstractMouse* createMouse() const override {
        return new DellMouse();
    }
};

// 苹果 (Apple) 家族工厂
class AppleFactory : public AbstractFactory {
public:
    AbstractKeyboard* createKeyboard() const override {
        return new AppleKeyboard();
    }
    AbstractMouse* createMouse() const override {
        return new AppleMouse();
    }
};
```

### 客户端(Client)

```cpp
void ClientCode(const AbstractFactory* factory) {
    // 客户端通过抽象接口获取一组产品
    AbstractKeyboard* keyboard = factory->createKeyboard();
    AbstractMouse* mouse = factory->createMouse();

    std::cout << "Client: Testing the produced family of devices..." << std::endl;
    
    // 使用产品
    keyboard->type();
    mouse->click();
    
    // 演示家族产品之间的协同工作
    mouse->workWith(keyboard); 
    
    // 清理内存
    delete keyboard;
    delete mouse;
}

// 示例主函数
void main() {
    std::cout << "--- 客户端：创建 Dell 家族设备 ---" << std::endl;
    AbstractFactory* dellFactory = new DellFactory();
    ClientCode(dellFactory);
    delete dellFactory;

    std::cout << "\n--- 客户端：创建 Apple 家族设备 ---" << std::endl;
    AbstractFactory* appleFactory = new AppleFactory();
    ClientCode(appleFactory);
    delete appleFactory;
}
```

### Output

```text
--- 客户端：创建 Dell 家族设备 ---
Client: Testing the produced family of devices...
Dell Keyboard: Typing with a focus on durability.
Dell Mouse: Clicking precisely.
Dell Mouse: Ensuring full compatibility with Dell Keyboard.

--- 客户端：创建 Apple 家族设备 ---
Client: Testing the produced family of devices...
Apple Keyboard: Typing with a sleek, low-profile design.
Apple Mouse: Clicking silently.
Apple Mouse: Optimized for macOS gesture control.
```

## 总结

- 如果没有应对“多系列对象构建”的需求变化，则没有必要使用Abstract Factory模式，这时候使用简单的工厂完全可以。
- “系列对象”指的是在某一特定系列下的对象之间有相互依赖、或作用的关系。不同系列的对象之间不能相互依赖。
- Abstract Factory模式主要在于应对“新系列”的需求变动。其缺点在于难以应对“新对象”的需求变动。
- **设计模式是解决稳定中有变化的场景。**
