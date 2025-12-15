---
layout: post
title: "Chain of Responsibility 职责链模式"
subtitle: ""
date: 2025-12-15
author: "Can"
header-img: "img/background/carnation.jpg"
tags: []
---

## Motivation

- 在软件构建过程中，一个请求可能会被多个对象处理，但是每个请求在运行时只能有一个接收者，如果显式指定，将必不可少地带来请求发送者与接收者的紧耦合。
- 如何使请求的发送者不需要指定具体的接受者呢？让请求的接受者自己在运行时决定来处理请求，从而使两者解耦。

## Defination

使多个对象都有机会处理请求，从而避免请求的发送者和接受者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止。

## Structure

![cor](/img/design-pattern/chain-of-responsibility.png)

## Example

这个示例将模拟一个采购审批流程，其中包含不同级别的审批者（如：经理、总监、CEO），他们根据采购金额来处理请求。

### 请求 (Request)

```cpp
// 采购请求类
class PurchaseRequest {
public:
    PurchaseRequest(int id, double amount, const string& purpose)
        : id_(id), amount_(amount), purpose_(purpose) {}

    double getAmount() const {
        return amount_;
    }

    int getId() const {
        return id_;
    }

    string getPurpose() const {
        return purpose_;
    }

private:
    int id_;      // 请求ID
    double amount_; // 采购金额
    string purpose_; // 采购目的
};
```

### 抽象处理器 (Handler)

```cpp
// 抽象审批者/处理器
class Approver {
public:
    // 设置下一个审批者（链中的下一个节点）
    void setNext(Approver* nextApprover) {
        nextApprover_ = nextApprover;
    }

    // 核心处理方法
    virtual void processRequest(const PurchaseRequest& request) = 0;

protected:
    Approver* nextApprover_ = nullptr; // 指向链中下一个处理器

    // 如果当前处理器无法处理，则转发给下一个处理器
    void forward(const PurchaseRequest& request) {
        if (nextApprover_) {
            nextApprover_->processRequest(request);
        } else {
            // 如果链条走到尽头，表示无人能处理
            cout << "---" << endl;
            cout << "请求ID " << request.getId() << " (金额: " << request.getAmount() << ") 无法被处理。" << endl;
        }
    }
};
```

### 具体处理器 (ConcreteHandler)

创建三个具体的审批者：Manager (经理)、Director (总监) 和 CEO (首席执行官)。

```cpp
// 经理 - 处理金额 <= 10000 的请求
class Manager : public Approver {
public:
    void processRequest(const PurchaseRequest& request) override {
        if (request.getAmount() <= 10000) {
            cout << "[经理] 审批通过: 请求ID " << request.getId() << " - 金额 " << request.getAmount() << "。" << endl;
        } else {
            cout << "[经理] 无法处理 请求ID " << request.getId() << " - 金额过高，转发给总监。" << endl;
            forward(request); // 转发给下一个
        }
    }
};

// 总监 - 处理金额 <= 50000 的请求
class Director : public Approver {
public:
    void processRequest(const PurchaseRequest& request) override {
        if (request.getAmount() <= 50000) {
            cout << "[总监] 审批通过: 请求ID " << request.getId() << " - 金额 " << request.getAmount() << "。" << endl;
        } else {
            cout << "[总监] 无法处理 请求ID " << request.getId() << " - 金额过高，转发给CEO。" << endl;
            forward(request); // 转发给下一个
        }
    }
};

// CEO - 处理金额 <= 100000 的请求
class CEO : public Approver {
public:
    void processRequest(const PurchaseRequest& request) override {
        if (request.getAmount() <= 100000) {
            cout << "[CEO] 最终审批通过: 请求ID " << request.getId() << " - 金额 " << request.getAmount() << "。" << endl;
        } else {
            cout << "[CEO] 无法处理 请求ID " << request.getId() << " - 拒绝 (金额超过最高权限)。" << endl;
            forward(request); // 转发给下一个 (这里会走到链条尽头)
        }
    }
};
```

### 客户端代码 (Client)

```cpp
// 客户端主函数
int main() {
    // 1. 创建处理器实例
    Manager* manager = new Manager();
    Director* director = new Director();
    CEO* ceo = new CEO();

    // 2. 构建责任链： 经理 -> 总监 -> CEO
    manager->setNext(director);
    director->setNext(ceo);

    // 3. 创建并发送请求

    // 请求1: 3000 元 (经理权限内)
    PurchaseRequest req1(1001, 3000, "办公用品");
    cout << "== 发送请求 1 ==" << endl;
    manager->processRequest(req1);
    cout << endl;

    // 请求2: 45000 元 (总监权限内)
    PurchaseRequest req2(1002, 45000, "新服务器");
    cout << "== 发送请求 2 ==" << endl;
    manager->processRequest(req2);
    cout << endl;

    // 请求3: 150000 元 (超过所有权限)
    PurchaseRequest req3(1003, 150000, "厂房购置");
    cout << "== 发送请求 3 ==" << endl;
    manager->processRequest(req3);
    cout << endl;
    
    // 4. 清理内存 (C++ 中需要手动或使用智能指针)
    delete manager;
    delete director;
    delete ceo;

    return 0;
}
```

### 运行结果

```text
== 发送请求 1 ==
[经理] 审批通过: 请求ID 1001 - 金额 3000。

== 发送请求 2 ==
[经理] 无法处理 请求ID 1002 - 金额过高，转发给总监。
[总监] 审批通过: 请求ID 1002 - 金额 45000。

== 发送请求 3 ==
[经理] 无法处理 请求ID 1003 - 金额过高，转发给总监。
[总监] 无法处理 请求ID 1003 - 金额过高，转发给CEO。
[CEO] 无法处理 请求ID 1003 - 拒绝 (金额超过最高权限)。
---
请求ID 1003 (金额: 150000) 无法被处理。
```

## Weakness

- 执行不确定性：请求沿着链条传递，如果到达链条末端仍没有处理器能够处理该请求，请求可能会被悄悄丢弃，客户端得不到明确的反馈（除非在链尾显式增加处理逻辑）。
- 性能和开销：每个请求都需要经过多个处理器的判断（if条件），导致多次方法调用和判断，增加了运行时开销；需要维护和实例化链条中的所有处理器对象。
- 设计复杂性：链条的结构通常是在运行时动态构建的。如果链条过长或顺序配置错误，会导致：
  - 调试困难：难以追踪请求是在哪一个处理器中被处理或停止的。
  - 配置复杂：客户端或配置系统需要负责正确地构建和维护链条的顺序。
- 代码耦合：处理器需要知道请求的具体类型和内部结构才能做出处理或转发的判断。如果请求的结构变化，所有相关的处理器都需要修改。
- 违背单一职责：在实际应用中，为了避免链条过长，开发者可能倾向于让一个处理器承担多个处理职责或复杂的判断逻辑，这违背了单一职责原则。

## Conclusion

- Chain of Responsibility模式的应用场合在于“一个请求可能有多个接受者，但是最后真正的接受者只有一个”，这时候请求发送者与接收者的耦合有可能出现“变化脆弱”的症状，职责链的目的就是将二者解耦，从而更好地应对变化。
- 应用该模式之后，对象的职责分派将更具灵活性，我们可以在运行时动态添加/修改请求的处理职责。
- 如果请求传递到职责链的末尾仍得不到处理，应该有一个合理的缺省机制。这也是每一个接受对象的职责，而不是发出请求的对象的责任。
