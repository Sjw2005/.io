# 01. Your first Behavior Tree

行为树与状态机类似，本质上只是在合适的时机和条件下调用**callbacks**的一种机制。这些回调函数内部执行什么操作，完全由你决定。

我们将交替使用“**to invoke the callback**”和**“to tick”**这两个表述。

在本教程系列中，我们的示例操作大多只是在控制台打印一些信息，但请记住，真正的“生产”代码通常会执行更复杂的操作。

接下来，我们将创建这个简单的树：

![image (3)](./assets/image%20(3)-1774278313753-11.png)

## **如何创建自己的 ActionNodes**

创建 TreeNode 的默认（且推荐）方法是通过继承。

```
// Example of custom SyncActionNode (synchronous action)
// without ports.
class ApproachObject : public BT::SyncActionNode
{
public:
  ApproachObject(const std::string& name) :
      BT::SyncActionNode(name, {})
  {}

  // You must override the virtual function tick()
  BT::NodeStatus tick() override
  {
    std::cout << "ApproachObject: " << this->name() << std::endl;
    return BT::NodeStatus::SUCCESS;
  }
};
```

如您所见：另外，我们也可以使用**依赖注入**，根据函数指针（即“函子”）来创建一个 TreeNode。

任何 TreeNode 实例都具有一个 `name`。该标识符旨在便于人类阅读，且**无需**具有唯一性。

方法 tick() 是实际执行操作的地方。它必须始终返回一个 NodeStatus，即 RUNNING、SUCCESS 或 FAILURE。

该 functor 必须具有以下签名：

```
BT::NodeStatus myFunction(BT::TreeNode& self)
```

例如：

```
using namespace BT;

// Simple function thatreturn a NodeStatus
BT::NodeStatus CheckBattery()
{
  std::cout << "[ Battery: OK ]" << std::endl;
  return BT::NodeStatus::SUCCESS;
}

// We want to wrap into an ActionNode the methods open() and close()
class GripperInterface
{
public:
  GripperInterface(): _open(true) {}

  NodeStatus open()
  {
        _open = true;
        std::cout << "GripperInterface::open" << std::endl;
        return NodeStatus::SUCCESS;
  }

  NodeStatus close()
  {
    std::cout << "GripperInterface::close" << std::endl;
        _open = false;
        return NodeStatus::SUCCESS;
  }

private:
  bool _open; // shared information
};
```

我们可以使用以下任何一个函子来构建一个 `SimpleActionNode`：

- CheckBattery()
- GripperInterface::open()
- GripperInterface::close()

## **使用 XML 动态创建树**

让我们来看一下名为 **my_tree.xml** 的以下 XML 文件：

```
 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
            <CheckBattery   name="check_battery"/>
            <OpenGripper    name="open_gripper"/>
            <ApproachObject name="approach_object"/>
            <CloseGripper   name="close_gripper"/>
        </Sequence>
     </BehaviorTree>
 </root>
```

有关 XML 模式的更多详细信息，请参见[此处](https://www.behaviortree.dev/docs/learn-the-basics/xml_format)。

我们必须先将自定义的 TreeNodes 注册到 `BehaviorTreeFactory` 中，然后从文件或文本中加载 XML。

XML 中使用的标识符必须与注册 TreeNodes 时使用的标识符一致。

“name” 属性表示实例的名称；**该属性是可选的**。

```
#include "behaviortree_cpp/bt_factory.h"

// file that contains the custom nodes definitions
#include "dummy_nodes.h"
using namespace DummyNodes;

int main()
{
    // We use the BehaviorTreeFactory to register our custom nodes
  BehaviorTreeFactory factory;

  // The recommended way to create a Node is through inheritance.
  factory.registerNodeType<ApproachObject>("ApproachObject");

  // Registering a SimpleActionNode using a function pointer.
  // You can use C++11 lambdas or std::bind
  factory.registerSimpleCondition("CheckBattery", [&](TreeNode&) { return CheckBattery(); });

  //You can also create SimpleActionNodes using methods of a class
  GripperInterface gripper;
  factory.registerSimpleAction("OpenGripper", [&](TreeNode&){ return gripper.open(); } );
  factory.registerSimpleAction("CloseGripper", [&](TreeNode&){ return gripper.close(); } );

  // Trees are created at deployment-time (i.e. at run-time, but only
  // once at the beginning).

  // IMPORTANT: when the object "tree" goes out of scope, all the
  // TreeNodes are destroyed
   auto tree = factory.createTreeFromFile("./my_tree.xml");

  // To "execute" a Tree you need to "tick" it.
  // The tick is propagated to the children based on the logic of the tree.
  // In this case, the entire sequence is executed, because all the children
  // of the Sequence return SUCCESS.
  tree.tickWhileRunning();

  return 0;
}

/* Expected output:
*
  [ Battery: OK ]
  GripperInterface::open
  ApproachObject: approach_object
  GripperInterface::close
*/
```