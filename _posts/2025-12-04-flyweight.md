---
layout: post
title: "Flyweight享元模式"
subtitle: ""
date: 2025-12-04
author: "Can"
header-img: "img/background/bridge.jpg"
tags: ["Design Pattern"]
---

## 动机

在软件系统采用纯粹对象方案的问题在于**大量细粒度的对象很快会充斥在系统中**，从而带来很高的运行代价——主要指**内存需求方面的代价**。如何在避免大量细粒度对象问题的同时，让外部客户程序仍然能够透明地使用面向对象的方式来操作呢？

## 定义

享元模式（Flyweight Pattern）是一种结构型设计模式，它主要用于减少对象的数量，从而提高系统的性能和效率。该模式通过**共享相同状态的对象**来实现这一目标，而不是为每个状态创建一个新的对象。

## 结构

![flyweight](/img/design-pattern/flyweight.png)

## 示例

以文本编辑器中的字符排版为例，在这个例子中：

- 内部状态（可共享的享元）： 字符的字体、大小、颜色等样式信息。
- 外部状态（不可共享的上下文）： 字符在文档中的位置、样式等信息。

### Flyweight 接口/抽象类

```cpp
// 抽象享元 (Flyweight) 接口
class CharacterStyle {
public:
    // display 方法需要外部状态 (x, y) 作为参数
    virtual void display(int x, int y) = 0;
    virtual ~CharacterStyle() {}
};
```

### Concrete Flyweight (内部状态：字体属性)

这是可共享的对象，它存储了所有具有相同样式的字符所需的数据。

```cpp
// 具体享元 (Concrete Flyweight) - 字体属性
class FontProperties : public CharacterStyle {
private:
    // 内部状态：共享属性
    string fontName;
    int fontSize;
    string fontColor;

public:
    FontProperties(string name, int size, string color) : 
        fontName(name), fontSize(size), fontColor(color) {
        // 模拟资源加载
        cout << "正在加载新的字体资源：[" << fontName << ", " 
             << fontSize << ", " << fontColor << "]" << endl;
    }

    // 实现 display 方法，使用内部状态和外部状态
    void display(int x, int y) override {
        // 使用共享的字体样式，结合外部位置信息显示字符
        cout << "  显示字符 (样式: " << fontName << ") 在 (" 
             << x << ", " << y << ")" << endl;
    }
    
    // 用于工厂查找的键
    string getKey() const {
        return fontName + "_" + to_string(fontSize) + "_" + fontColor;
    }
};
```

### Flyweight Factory (享元工厂)

享元工厂负责创建和管理享元对象。它确保在需要时返回一个已存在的对象，或者创建一个新的对象。

```cpp
// 享元工厂 (Flyweight Factory)
class StyleFactory {
private:
    // 使用样式组合作为 Key，存储已创建的共享样式对象
    map<string, FontProperties*> sharedStyles;

public:
    // 获取享元对象的关键方法
    FontProperties* getStyle(string name, int size, string color) {
        string key = name + "_" + to_string(size) + "_" + color;
        
        // 1. 检查是否已存在具有此样式的享元
        if (sharedStyles.find(key) == sharedStyles.end()) {
            // 2. 如果不存在，则创建新的享元对象并存储
            cout << "--- 工厂创建新的共享字体样式 ---" << endl;
            sharedStyles[key] = new FontProperties(name, size, color);
        }
        
        // 3. 返回已有的或新创建的对象
        return sharedStyles[key];
    }

    // 清理资源
    ~StyleFactory() {
        for (auto const& [key, val] : sharedStyles) {
            delete val;
        }
        sharedStyles.clear();
    }
};
```

### Client (客户端/上下文：文档中的单个字符)

客户端负责持有对享元对象的引用，并管理所有字符独有的外部状态（位置和字符本身）。

```cpp
// 客户端/上下文 (Client/Context) - 文档中的一个字符实例
class DocumentCharacter {
private:
    char value;                 // 字符值
    FontProperties* sharedStyle; // 内部状态：共享的样式对象
    
    // 外部状态：不可共享，每个字符独有
    int x; 
    int y; 

public:
    DocumentCharacter(char c, FontProperties* style, int posX, int posY) : 
        value(c), sharedStyle(style), x(posX), y(posY) {}

    void render() {
        cout << "字符 '" << value << "':";
        // 渲染时将外部状态传递给享元对象，享元对象负责处理内部状态
        sharedStyle->display(x, y); 
    }
};

// --- 主程序 (Main function) ---
void main() {
    StyleFactory factory;
    vector<DocumentCharacter> document;
    
    // **场景 1: 标题 (粗体，20号，红色)**
    FontProperties* titleStyle = factory.getStyle("Arial", 20, "Red");
    
    // 假设字符 'H', 'e', 'l', 'l', 'o'
    document.emplace_back('H', titleStyle, 10, 10); // X:10, Y:10
    document.emplace_back('e', titleStyle, 20, 10); 
    document.emplace_back('l', titleStyle, 30, 10);
    document.emplace_back('l', titleStyle, 40, 10);
    document.emplace_back('o', titleStyle, 50, 10);

    // **场景 2: 正文 (普通，12号，黑色)**
    FontProperties* bodyStyle = factory.getStyle("Arial", 12, "Black");

    // 假设字符 'W', 'o', 'r', 'l', 'd'
    document.emplace_back('W', bodyStyle, 10, 30); // X:10, Y:30
    document.emplace_back('o', bodyStyle, 20, 30);
    
    // 即使正文有成千上万个字符，它们都只引用同一个 bodyStyle 享元对象。

    // **场景 3: 另一个标题 (粗体，20号，红色)**
    // 再次调用 factory.getStyle，会返回**之前创建的** titleStyle 对象，不会重复创建
    FontProperties* anotherTitleStyle = factory.getStyle("Arial", 20, "Red");
    
    cout << "\n--- 渲染文档 ---\n";
    for (DocumentCharacter& c : document) {
        c.render();
    }
    cout << "--- 渲染完成 ---\n";
    
    // 总结: 
    // 1. 文档中有 7 个 DocumentCharacter 对象。
    // 2. 内存中只创建了 2 个 FontProperties 享元对象 (titleStyle 和 bodyStyle)。
}
```

这个例子清晰地展示了，即使系统中有大量字符实例，但只要它们的样式（内部状态）相同，它们就会共享同一个享元对象，从而节省了存储大量重复样式信息的内存。

### flyweight vs prototype

- **享元模式**：
  - 享元对象在设计上就是为了**多线程安全**而生的，因为它们的核心是**不可变性**。
  - 享元模式将对象的状态分为**内部状态（Intrinsic State，共享的部分，如字体名称、颜色）**和**外部状态（Extrinsic State，不共享的部分，如坐标）**。内部状态一旦被创建，通常就是固定不变的（只读）。多线程可以同时访问和读取这些不可变的状态，不会造成任何线程冲突（即不会发生竞态条件/Race Condition）。即**写操作需要控制并发，因为可能会导致脏读、不可重复读等问题**。
- **原型模式**：
  - 原型模式的目的在于通过克隆（clone()）来创建新的、独立的对象实例。这些被克隆的对象通常是**可变的**。
  - 克隆出来的对象是**完全独立的新实例**，它们将由各自的线程独立地修改和管理。但为了**确保克隆得到的初始状态是有效的、完整的**，对原始原型对象进行操作时，就必须考虑**并发控制**。

## 总结

- 面向对象很好地解决了抽象性的问题，但是作为一个运行在机器中的程序实体，我们需要考虑**对象的代价问题**。Flyweight主要解决面向对象的代价问题，一般不触及面向对象的抽象性问题。
- Flyweight采用**对象共享**的做法来降低系统中对象的个数，从而降低细粒度对象给系统带来的内存压力。
- 对象的数量大小需要根据具体的应用进行评估，而不能凭空臆断。
