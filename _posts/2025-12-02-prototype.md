---
layout: post
title: "Prototype原型模式"
subtitle: ""
date: 2025-12-02
author: "Can"
header-img: "img/sunflower-1.jpg"
tags: ["Design Pattern"]
---

## 模式定义

使用原型实例指定创建对象的种类，然后通过拷贝这些原型来创建新的对象。

## 模式结构

![Prototype模式结构](/img/design-pattern/prototype.png)

- **Prototype**：原型类，声明了一个克隆自身的接口。
- **ConcretePrototype**：具体原型类，实现了克隆自身的接口，定义了一个可以克隆自己的方法。
- **Client**：客户端类，使用原型类来创建新的对象。

## 示例

### 抽象原型类(Prototype)

```cpp
class AbstractDocumentPrototype {
public:
    virtual ~AbstractDocumentPrototype() = default;

    // 声明克隆自身的纯虚方法, 返回一个指向新创建对象的指针
    virtual AbstractDocumentPrototype* clone() const = 0;

    // 其他操作/方法
    virtual void printContent() const = 0;
};
```

### 具体原型类(ConcretePrototype)

```cpp
// 具体原型类：例如，一份“报告”文档
class ReportDocument : public AbstractDocumentPrototype {
private:
    std::string title;
    int pageCount;
    // 其他复杂数据成员...

public:
    // 构造函数
    ReportDocument(const std::string& t, int pc) 
        : title(t), pageCount(pc) {}

    // 拷贝构造函数：实现深拷贝的关键
    ReportDocument(const ReportDocument& other) 
        : AbstractDocumentPrototype(other) // 伪代码：调用基类拷贝构造
    {
        // 实现深拷贝：复制所有数据成员
        this->title = other.title;
        this->pageCount = other.pageCount;
        // 如果有指针或资源，需在此处创建新资源并复制数据
    }

    // 实现 clone() 方法
    AbstractDocumentPrototype* clone() const override {
        // 使用拷贝构造函数来创建对象的一个精确副本
        // 在 C++ 中，这通常是通过调用 'new ConcretePrototype(*this)' 来实现
        return new ReportDocument(*this);
    }

    // 实现其他方法
    void printContent() const override {
        std::cout << "Document Type: Report \n";
        std::cout << "Title: " << title << " \n";
        std::cout << "Pages: " << pageCount << " \n";
    }

    // 修改数据的方法 (用于演示克隆后的独立性)
    void changeTitle(const std::string& newTitle) {
        title = newTitle;
    }
};
```

### 客户端代码（Client Code）

```cpp
// 客户端代码示例
void clientCode() {
    // 1. 创建一个具体的原型对象 (Original Prototype)
    // 这是一个代价较高的、复杂的或配置好的对象
    ReportDocument* originalReport = new ReportDocument("2024 Annual Financial Report", 50);
    std::cout << "--- Original Document ---\n";
    originalReport->printContent();
    std::cout << "Address: " << originalReport << "\n\n";

    // 2. 通过调用 clone() 方法创建新对象 (Cloned Prototype)
    ReportDocument* draftReport = dynamic_cast<ReportDocument*>(originalReport->clone());
    
    // 3. 验证新对象和原对象是独立的
    std::cout << "--- Cloned Draft Document ---\n";
    draftReport->changeTitle("2024 Annual Financial Report - Draft v1.0"); // 修改副本
    draftReport->printContent();
    std::cout << "Address: " << draftReport << "\n\n";

    // 4. 再次查看原对象，确认它没有被修改
    std::cout << "--- Original Document (After Clone) ---\n";
    originalReport->printContent();
    std::cout << "Address: " << originalReport << "\n\n";
    
    // 清理
    delete originalReport;
    delete draftReport;
}
```

## 原型模式的优势（相对于new操作）

- **降低创建复杂对象的成本（或开销）**：new操作创建对象的过程中涉及到大量的资源消耗：1）创建对象需要从数据库加载大量配置数据或读取大文件。2）对象的构造函数中包含复杂的、耗时的计算或初始化逻辑。3）可能需要进行网络连接或与外部设备通信。而对于clone()来说，**一旦第一个“原型”对象被创建并初始化，后续的新对象只需要复制内存中的现有状态即可**。这通常比重新执行复杂的初始化步骤要快得多。
- **避免与特定类耦合**：通过使用原型模式，客户端代码可以独立于具体的类实现。这意味着如果需要创建新的对象类型，只需要实现新的原型类，而不需要修改客户端代码。这符合“**开闭原则**”，即对扩展是开放的，对修改是封闭的。
- **动态配置对象**：原型模式允许在运行时动态地配置对象的属性。这对于需要根据不同场景创建不同配置的对象非常有用。例如，在游戏开发中，可能需要根据玩家选择的不同游戏模式（如普通模式、困难模式等）创建不同的游戏角色。

## 解耦原理

原型模式的核心在于引入一个**原型管理器（Prototype Manager）或注册表（Registry）**。这个管理器负责持有所有可供克隆的原型实例。当客户端代码需要一个对象时，它不直接 new，而是向管理器请求一个已有的原型实例，并调用这个实例的克隆方法 (clone())。_通过操作已有的原型实例（接口）和抽象标识符，而不是通过直接调用构造函数（具体类名）来创建对象，从而实现客户端代码与具体实现类的解耦。_

```java
// 1. 定义原型接口
interface Prototype {
    Prototype clone(); // 客户端只知道这个方法
}

// 2. 原型管理器（通常由框架或服务器端初始化）
class PrototypeManager {
    // 存储原型对象实例，使用字符串键来引用它们
    private Map<String, Prototype> prototypes = new HashMap<>(); 

    public Prototype getPrototype(String name) {
        // 客户端通过名称（而非类名）请求对象
        return prototypes.get(name).clone(); // 客户端调用 clone()
    }
    // ... 注册原型的方法 ...
}

// 3. 客户端代码
class Client {
    private PrototypeManager manager;

    public void createProduct() {
        // 客户端只知道它想要一个名为 "A" 的产品
        // 客户端不知道 "A" 实际是 ConcreteProductA 这个类
        Prototype product = manager.getPrototype("A"); 
        
        // 客户端只需要知道原型接口的方法
        product.doStuff(); 
    }
}
```

## 总结

- 原型模式是为了解决创建**昂贵或复杂对象**的效率问题，而不是为了代替创建简单对象。
- Prototype模式同样用于**隔离类对象的使用者和具体类型（易变类）之间的耦合关系**，它同样要求这些“易变类”拥有“稳定的接口”。
- Prototype模式对于*“**如何创建易变类的实体对象**”*采用“**原型克隆**”的方法来做，它使得我们可以非常灵活地动态创建“拥有某些稳定接口”的新对象——所需工作仅仅只是注册一个新类的对象（即原型），然后在任何需要的地方clone。
- Prototype模式中的clone方法可以利用某些框架中的序列化来实现**深拷贝**。
