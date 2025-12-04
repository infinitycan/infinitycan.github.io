---
layout: post
title: "Decorator装饰模式"
subtitle: ""
date: 2025-10-16
author: "Can"
header-img: "img/nature-3.jpg"
tags: ["Design Pattern"]
---

## 定义

动态**（组合）** 地给一个对象 **增加一些额外的职责**。就增加功能而言，Decorator模式比生成子类（继承）更为灵活**（消除重复代码 & 减少子类个数）**。

## 结构

![Decorator](/img/in-post/decorator.png)

## 示例

1. 结构定义（Component接口）

   ```cpp
   class Beverage {
   public:
       // 纯虚函数定义统一接口
       virtual double getCost() const = 0;
       virtual std::string getDescription() const = 0;
       virtual ~Beverage() {}
   };
   ```

2. 基本组件（Concrete Component - 被装饰者）

   ```cpp
   class Coffee : public Beverage {
   public:
       double getCost() const override {
           return 5.0; // 基础咖啡价格
       }
   
       std::string getDescription() const override {
           return "纯咖啡";
       }
   };
   ```

3. 抽象装饰者（Decorator）

   ```cpp
   class CondimentDecorator : public Beverage { // <-- 继承 Beverage
   protected:
       // 组合关系：持有对被包装对象的引用
       // 关键：类型是抽象的 Beverage*，实现多态
       Beverage* beverage; 
   
   public:
       // 构造函数，接受一个被包装对象
       CondimentDecorator(Beverage* b) : beverage(b) {}
       
       // 抽象装饰者仍保持抽象，强制子类实现具体的装饰逻辑
       // 实际项目中，有时会在此处实现委派，但为了清晰展示装饰逻辑，我们保持它抽象
       // virtual double getCost() const = 0;
       // virtual std::string getDescription() const = 0;
   
       // 注意：这里的析构函数需要考虑谁来释放 beverage 对象
       // 实际项目中应使用 std::unique_ptr 来安全管理
       virtual ~CondimentDecorator() {
           // 简化起见，此处不处理所有权和释放
       }
   };
   ```

4. 具体装饰者（Concrete Decorator）

   ```cpp
   class Milk : public CondimentDecorator {
   public:
       Milk(Beverage* b) : CondimentDecorator(b) {}
   
       std::string getDescription() const override {
           // 装饰者逻辑：多态调用被包装者的描述 + 自己的描述
           return beverage->getDescription() + ", 加牛奶"; 
       }
   
       double getCost() const override {
           // 装饰者逻辑：多态调用被包装者的价格 + 自己的价格
           return beverage->getCost() + 2.0; // 2.0是牛奶的价格
       }
   };
   
   ```

   

5. 客户端代码（运行时实现多态）

   ```cpp
   void clientCode() {
       // 1. 创建基本组件 (纯咖啡)
       Beverage* myCoffee = new Coffee(); 
   
       std::cout << "初始: " << myCoffee->getDescription() << ", 价格: " << myCoffee->getCost() << std::endl;
   
       // 2. 第一次装饰：加牛奶 (Milk 包装 Coffee)
       // 此时 myCoffee 变量指向 Milk 对象
       myCoffee = new Milk(myCoffee); 
   
       std::cout << "装饰后: " << myCoffee->getDescription() << ", 价格: " << myCoffee->getCost() << std::endl;
       // 调用 myCoffee->getCost() 时，多态链如下：
       // Milk::getCost() -> beverage->getCost() (调用 Coffee::getCost()) + 2.0 = 5.0 + 2.0 = 7.0
   
       // 3. (假设有 Mocha 类) 第二次装饰：加摩卡 (Mocha 包装 Milk)
       // myCoffee = new Mocha(myCoffee); 
       
       // 资源清理（裸指针需要手动清理，实际项目务必使用智能指针）
       delete myCoffee; 
   }
   ```

## 总结

- Decorator模式实现了在运行时动态扩展对象功能的能力，而且可以根据需要扩展多个功能。避免了使用继承带来的“灵活性差”和“多子类衍生的问题”。
- Decorator类在接口上表现为is-a Component的继承关系，即Decorator类继承了Component类所具有的结构。但在实现上又表现为has-a Component的组合关系，即Decorator类又使用了另外的一个Component类。
- Decorator模式的目的并非要解决“多子类衍生的多继承”问题，而是在于解决“主体类在多个方向上的扩展功能”——是为“装饰”的含义。
- Decorator模式的显著特点是：**抽象装饰者（Decorator）继承抽象组件（Component），以确保装饰者和被装饰者拥有相同的接口（实现类型透明）；同时，抽象装饰者（Decorator）组合一个抽象组件（Component）的引用，以实现对被包装对象的多态调用和行为增强。**
