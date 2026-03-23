# 08. Pass additional arguments

## Pass additional arguments to your Nodes

向节点传递额外参数

在我们迄今为止探讨的每一个示例中，我们都“被迫”提供一个具有以下签名的构造函数

```
MyCustomNode(const std::string& name, const NodeConfig& config);
```

在某些情况下，我们可能需要向类的构造函数传递额外的参数、指针、引用等。

有些人会用黑板来做这件事。别这么做。

在本教程的剩余部分，我们将统一使用“*arguments*”一词。

**即使从理论上讲，可以通过输入端口传递这些参数，但在以下情况下，这仍是一种错误的做法：**

- 这些参数在部署时（构建树时）已知。
- 这些参数在运行时不会改变。
- 这些参数无需通过 XML 进行设置。

**如果满足所有这些条件，则强烈建议不要使用端口或黑板。**

## 在构造函数中添加参数（推荐）

请考虑以下名为 Action_A 的自定义节点。

我们希望传递两个额外的参数；它们可以是任意复杂的对象，并不局限于内置类型。

```
// Action_A has a different constructor than the default one.
class Action_A: public SyncActionNode
{

public:
    // additional arguments passed to the constructor
    Action_A(const std::string& name, const NodeConfig& config,
             int arg_int, std::string arg_str):
        SyncActionNode(name, config),
        _arg1(arg_int),
        _arg2(arg_str) {}

    // this example doesn't require any port
    static PortsList providedPorts() { return {}; }

    // tick() can access the private members
    NodeStatus tick() override;

private:
    int _arg1;
    std::string _arg2;
};
```

**注册此节点并传入已知的参数非常简单，只需：**

```
BT::BehaviorTreeFactory factory;
factory.registerNodeType<Action_A>("Action_A", 42, "hello world");

// If you prefer to specify the template parameters
// factory.registerNodeType<Action_A, int, std::string>("Action_A", 42, "hello world");
```

## 使用“initialize”方法

如果出于某种原因，您需要向 Node 类型的各个实例传递不同的值，不妨考虑以下另一种模式：

```
class Action_B: public SyncActionNode
{

public:
    // The constructor looks as usual.
    Action_B(const std::string& name, const NodeConfig& config):
        SyncActionNode(name, config) {}

    // We want this method to be called ONCE and BEFORE the first tick()
    void initialize(int arg_int, const std::string& arg_str)
    {
        _arg1 = arg_int;
        _arg2 = arg_str;
    }

    // this example doesn't require any port
    static PortsList providedPorts() { return {}; }

    // tick() can access the private members
    NodeStatus tick() override;

private:
    int _arg1;
    std::string _arg2;
};
```

**我们注册和初始化 Action_B 的方式有所不同：**

```
BT::BehaviorTreeFactory factory;

// Register as usual, but we still need to initialize
factory.registerNodeType<Action_B>("Action_B");

// Create the whole tree. Instances of Action_B are not initialized yet
auto tree = factory.createTreeFromText(xml_text);

// visitor will initialize the instances of
auto visitor = [](TreeNode* node)
{
  if (auto action_B_node = dynamic_cast<Action_B*>(node))
  {
    action_B_node->initialize(69, "interesting_value");
  }
};

// Apply the visitor to ALL the nodes of the tree
tree.applyVisitor(visitor);
```