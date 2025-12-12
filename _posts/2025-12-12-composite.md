---
layout: post
title: "Composite组合模式"
subtitle: ""
date: 2025-12-12
author: "Can"
header-img: "img/background/hemlock.jpg"
tags: ["Design Pattern"]
---

## Motivation

- 在软件的某种情况下，客户代码过多地依赖于对象容器的复杂内部实现结构，对象容器内部实现结构（而非抽象接口）的变化将引起客户代码的频繁变化，带来代码的维护性、扩展性等弊端。
- 如何将“客户代码与复杂的对象容器结构”解耦？让对象容器自己来实现自身的复杂结构，从而使得客户代码就像处理简单对象一样来处理复杂的对象容器？

## Definition

将对象组合成树形结构以表示“部分-整体”的层次结构。Composite使得用户对单个对象和组合对象的使用具有一致性（稳定）。

## Structure

![composite](/img/design-pattern/composite.png)

## Example

### 抽象组件（Component）

这是所有叶子和组合节点的公共基类，定义了它们共享的接口。

```cpp
/**
 * @brief 抽象组件（Component）
 * 为组合中的对象声明接口。
 */
class Component {
public:
    virtual ~Component() = default;

    /**
     * @brief 客户端操作，所有组件都必须实现。
     * 例如：显示、计算成本等。
     */
    virtual void operation() = 0;

    // --- 组合特有的方法 (Leaf类不需要，但为了统一接口通常放在这里) ---

    /**
     * @brief 添加子组件（主要用于 Composite 类）
     */
    virtual void add(Component* component) {
        // 默认实现为空或抛出异常，因为 Leaf 节点不支持添加子组件。
        // 在 C++ 中，通常将此方法移至 Composite 类以实现安全性和透明性的平衡。
        // 但为保持统一接口，此处保留。
    }

    /**
     * @brief 移除子组件
     */
    virtual void remove(Component* component) {
        // 默认实现为空
    }

    /**
     * @brief 获取子组件
     */
    virtual Component* getChild(int index) {
        return nullptr; // 默认返回空
    }
};
```

### 叶子（Leaf）

叶子是组合的末端对象，不包含其他组件。

```cpp
/**
 * @brief 叶子（Leaf）
 * 代表组合中的叶子对象，没有子对象。
 */
class Leaf : public Component {
public:
    std::string name;

    Leaf(const std::string& n) : name(n) {}

    /**
     * @brief 实现客户端操作
     */
    void operation() override {
        std::cout << "Leaf " << name << ": 执行操作" << std::endl;
    }

    // Leaf 类通常不实现 add/remove/getChild，因为它不能有子组件。
    // 它从 Component 继承了这些方法，但其实现要么为空，要么抛出异常。
};
```

### 组合节点（Composite）

组合节点可以包含叶子或其他组合节点（子组件）。

```cpp
/**
 * @brief 组合节点（Composite）
 * 定义有子组件的组件的行为，并存储子组件。
 */
class Composite : public Component {
private:
    std::vector<Component*> children;
    std::string name;

public:
    Composite(const std::string& n) : name(n) {}

    ~Composite() {
        // 释放子组件的内存
        for (Component* child : children) {
            delete child;
        }
    }

    /**
     * @brief 递归地实现客户端操作
     */
    void operation() override {
        std::cout << "Composite " << name << ": 开始执行操作..." << std::endl;
        for (Component* child : children) {
            // 递归调用子组件的 operation()
            child->operation();
        }
        std::cout << "Composite " << name << ": 操作完成。" << std::endl;
    }

    /**
     * @brief 添加子组件
     */
    void add(Component* component) override {
        children.push_back(component);
    }

    /**
     * @brief 移除子组件
     */
    void remove(Component* component) override {
        // 伪代码：实际实现需要查找并移除特定组件
        // children.erase(std::remove(children.begin(), children.end(), component), children.end());
    }

    /**
     * @brief 获取子组件
     */
    Component* getChild(int index) override {
        if (index >= 0 && index < children.size()) {
            return children[index];
        }
        return nullptr;
    }
};
```

### 客户端代码（Client）

客户端代码可以统一处理叶子对象和组合对象。

```cpp
/**
 * @brief 客户端代码（Client）
 */
void clientCode(Component* component) {
    // 客户端无需区分传入的是 Leaf 还是 Composite，因为它们共享 Component 接口。
    component->operation();
}

void main() {
    // 1. 创建叶子对象
    Leaf* file1 = new Leaf("file_a.txt");
    Leaf* file2 = new Leaf("file_b.doc");
    Leaf* file3 = new Leaf("image_c.jpg");

    // 2. 创建组合对象（目录/文件夹）
    Composite* folder1 = new Composite("Documents");
    Composite* folder2 = new Composite("Images");
    Composite* root = new Composite("RootFolder");

    // 3. 组合对象（构建树形结构）
    // Documents 包含 file1 和 file2
    folder1->add(file1);
    folder1->add(file2);

    // Images 包含 file3
    folder2->add(file3);

    // RootFolder 包含 Documents 和 Images
    root->add(folder1);
    root->add(folder2);

    // 4. 客户端调用：统一处理
    std::cout << "--- 调用 RootFolder (组合对象) 的 operation ---" << std::endl;
    clientCode(root);

    std::cout << "\n--- 调用 file1 (叶子对象) 的 operation ---" << std::endl;
    clientCode(file1);
    
    // 在 Composite 的析构函数中已经包含了子组件的删除。
    delete root; // 删除根节点将自动删除所有子节点（file1, file2, folder1, folder2, file3）
}
```

## Conclusion

- Composite模式采用**树形结构**来实现普遍存在的对象容器，从而**将“一对多”的关系转换为“一对一”的关系**，使得客户代码可以一致地（**复用**）处理对象和对象容器，无需关心处理的是单个对象，还是组合的对象容器。
- 将“客户代码与复杂的对象容器结构”**解耦**是Composite的核心思想，解耦之后，**客户端代码将与纯粹的抽象接口——而非对象容器的内部实现结构——发生依赖**，从而更能“应对变化”。
- Composite模式在具体实现中，可以让父对象中的子对象**反向追溯**；如果父对象有频繁的遍历需求，可使用**缓存**技巧来改善效率。
