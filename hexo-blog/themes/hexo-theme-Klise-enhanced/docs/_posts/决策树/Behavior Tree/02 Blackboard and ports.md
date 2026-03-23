# 02. Blackboard and ports

正如我们之前所解释的，自定义 TreeNodes 可以用来执行任意简单或复杂的软件代码。其目的是提供一个具有**更高抽象层次**的接口。

因此，从概念上讲，它们与**函数**并无二致。

与函数类似，我们通常希望：BehaviorTree.CPP 提供了一种通过 **端口** 进行 **数据流** 的基本机制，这种机制不仅易于使用，而且灵活且类型安全。

向节点传递参数（**输入**）

从节点获取某种信息（输出）。一个节点的输出可以作为另一个节点的输入。

在本教程中，我们将创建以下树：

![image (4)](./assets/image%20(4)-1774278334808-14.png)

- “黑板”是一种由树中所有节点共享的简单**键值存储**。
- 黑板中的一个“条目”是一个**键值对**。
- **输入端口**可以读取黑板中的条目，而**输出端口**可以向条目写入数据。

## **输入端口**

有效的输入可以是以下任一种：假设我们要创建一个名为 `SaySomething` 的 ActionNode，它应在 `std::cout` 上打印给定的字符串。

- 一个 **静态字符串**，该节点将读取并解析它；或者
- 指向黑板中某条目的“指针”，该条目由一个 **键** 标识。

为了传递这个字符串，我们将使用一个名为 **message** 的输入端口。

请考虑以下这些替代的 XML 语法：

```
<SaySomething name="first"   message="hello world" />
<SaySomething name="second" message="{greetings}" />
```

- 在**第一个**节点中，端口接收字符串“hello world”；
- 而第二个节点则被要求使用条目“greetings”在黑板上查找该值。

条目“greetings”的值在运行时可能会（而且很可能）发生变化。

ActionNode `SaySomething`可以按以下方式实现：

```
// SyncActionNode (synchronous action) with an input port.
class SaySomething : public SyncActionNode
{
public:
  // If your Node has ports, you must use this constructor signature
  SaySomething(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }

  // It is mandatory to define this STATIC method.
  static PortsList providedPorts()
  {
    // This action has a single input port called "message"
    return { InputPort<std::string>("message") };
  }

  // Override the virtual function tick()
  NodeStatus tick() override
  {
    Expected<std::string> msg = getInput<std::string>("message");
    // Check if expected is valid. If not, throw its error
    if (!msg)
    {
      throw BT::RuntimeError("missing required input [message]: ",
                              msg.error() );
    }
    // use the method value() to extract the valid message.
    std::cout << "Robot says: " << msg.value() << std::endl;
    return NodeStatus::SUCCESS;
  }
};
```

当自定义 TreeNode 具有输入和/或输出端口时，必须在 **static** 方法中声明这些端口：

```
static MyCustomNode::PortsList providedPorts();
```

可以通过模板方法 `TreeNode::getInput<T>(key)`读取端口 `message`的输入。

该方法可能因多种原因失败。用户需自行检查返回值的有效性，并决定如何处理：

- 返回 `NodeStatus::FAILURE`？
- 抛出异常？
- 使用其他默认值？

**始终**建议在`tick()`内部调用`getInput()`方法，**而非**在类的构造函数中调用。

C++ 代码应预期输入的实际值会在**运行时发生变化**，因此应定期对其进行更新。

## **输出端口**

指向黑板条目（entry）的输入端口仅在另一个节点已在该条目中写入“某些内容”时才有效。

`ThinkWhatToSay`是一个使用**输出端口**将字符串写入条目的节点示例。

```
class ThinkWhatToSay : public SyncActionNode
{
public:
  ThinkWhatToSay(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }

  static PortsList providedPorts()
  {
    return { OutputPort<std::string>("text") };
  }

  // This Action writes a value into the port "text"
  NodeStatus tick() override
  {
    // the output may change at each tick(). Here we keep it simple.
    setOutput("text", "The answer is 42" );
    return NodeStatus::SUCCESS;
  }
};
```

此外，在大多数情况下，为了调试目的，可以使用名为 `Script` 的内置操作将静态值写入条目中。

```
<Script code=" the_answer:=‘The answer is 42’ " />
```

我们将在关于 [BT.CPP 中的新脚本语言](https://www.behaviortree.dev/docs/guides/scripting) 的教程中进一步探讨 **Script** 操作

如果您是从 BT.CPP 3.X 迁移而来，Script 可以直接替代现已不建议使用的 **SetBlackboard**。

## **双向端口**

有时，节点需要同时**读取和修改**黑板上的某个值。在这种情况下，请使用`BidirectionalPort<T>`，它结合了InputPort和OutputPort的功能。

```
static PortsList providedPorts()
{
    return { BidirectionalPort<std::vector<int>>("vector"),
             InputPort<int>("value") };
}
```

双向端口通常与 `getLockedPortContent()` 配合使用，以实现对黑板条目的线程安全读取-修改-写入访问。完整的说明和示例请参见 [教程 13：通过引用访问端口](https://www.behaviortree.dev/docs/tutorial-advanced/tutorial_13_blackboard_reference)。

## **完整示例**

在此示例中，将执行由 3 个操作组成的序列：

- 操作 1 从静态字符串中读取输入 `message`。
- 操作 2 将内容写入名为 `the_answer`的黑板条目中。
- 操作 3 从名为 `the_answer`的黑板条目中读取输入 `message`。

```
<root BTCPP_format="4" >
    <BehaviorTree ID="MainTree">
       <Sequence name="root_sequence">
           <SaySomething     message="hello" />
           <ThinkWhatToSay   text="{the_answer}"/>
           <SaySomething     message="{the_answer}" />
       </Sequence>
    </BehaviorTree>
</root>
```

用于注册和执行树的 C++ 代码：

```
#include "behaviortree_cpp/bt_factory.h"

// file that contains the custom nodes definitions
#include "dummy_nodes.h"
using namespace DummyNodes;

int main()
{
  BehaviorTreeFactory factory;
  factory.registerNodeType<SaySomething>("SaySomething");
  factory.registerNodeType<ThinkWhatToSay>("ThinkWhatToSay");

  auto tree = factory.createTreeFromFile("./my_tree.xml");
  tree.tickWhileRunning();
  return 0;
}

/*  Expected output:
  Robot says: hello
  Robot says: The answer is 42
*/
```

我们使用相同的键（`the_answer`）将输出端口与输入端口“连接”起来；换句话说，它们“指向”黑板中的同一条记录。

这些端口之所以可以相互连接，是因为它们的类型相同，即 `std::string`。如果你尝试连接类型不同的端口，`factory.createTreeFromFile` 方法将会抛出异常。