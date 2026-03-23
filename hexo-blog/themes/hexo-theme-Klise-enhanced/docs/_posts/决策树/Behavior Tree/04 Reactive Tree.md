# 04. Reactive Tree

## 响应式与异步行为

下面的示例展示了 SequenceNode 和 ReactiveSequence 之间的区别。

我们将实现一个异步操作，即一个需要较长时间才能完成的操作，且在满足完成条件之前会返回 RUNNING 状态。

**异步操作需满足以下要求：**

- 它不应在 tick() 方法中阻塞过长时间。应尽快恢复执行流程。
- 如果调用了 halt() 方法，应尽快中止该操作。

深入了解异步操作:

用户应充分理解 BT.CPP 中如何实现并发，并学习开发自有异步操作的最佳实践。您可以在这里找到一篇详尽的文章。

## StatefulActionNode

StatefulActionNode 是实现异步操作的首选方式。当您的代码包含请求-响应模式时，它特别有用，即当操作向另一个进程发送异步请求，并定期检查是否已收到响应时。

**根据该响应，它可能会返回 SUCCESS 或 FAILURE。**

如果不是与外部进程通信，而是执行耗时较长的计算，您可以将其拆分为多个小“块”，或者将该计算移至另一个线程（参见 AsyncThreadedAction 教程）。

StatefulActionNode 的派生类必须重写以下虚拟方法，而不是 `tick()`：

- NodeStatus onStart()：当节点处于 IDLE 状态时调用。该方法可能立即成功或失败，也可能返回 RUNNING 状态。在后一种情况下，下次接收到 tick 时将执行 onRunning 方法。
- NodeStatus onRunning()：当节点处于 RUNNING 状态时调用。返回新的状态。
- void onHalted()：当该节点被树中的另一个节点中止时调用。

### 让我们创建一个名为 MoveBaseAction 的虚拟节点：

```
// Custom type
struct Pose2D
{
    double x, y, theta;
};

namespace chr = std::chrono;

class MoveBaseAction : public BT::StatefulActionNode
{
  public:
    // Any TreeNode with ports must have a constructor with this signature
    MoveBaseAction(const std::string& name, const BT::NodeConfig& config)
      : StatefulActionNode(name, config)
    {}

    // It is mandatory to define this static method.
    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<Pose2D>("goal") };
    }

    // this function is invoked once at the beginning.
    BT::NodeStatus onStart() override;

    // If onStart()returned RUNNING, we will keep calling
    // this method until it return something different from RUNNING
    BT::NodeStatus onRunning() override;

    // callback to execute if the action was aborted by another node
    void onHalted() override;

  private:
    Pose2D _goal;
    chr::system_clock::time_point _completion_time;
};

//-------------------------

BT::NodeStatus MoveBaseAction::onStart()
{
  if ( !getInput<Pose2D>("goal", _goal))
  {
    throw BT::RuntimeError("missing required input [goal]");
  }
  printf("[ MoveBase: SEND REQUEST ]. goal: x=%f y=%f theta=%f\n",
         _goal.x, _goal.y, _goal.theta);

  // We use this counter to simulate an action that takes a certain
  // amount of time to be completed (200 ms)
  _completion_time = chr::system_clock::now() + chr::milliseconds(220);

  return BT::NodeStatus::RUNNING;
}

BT::NodeStatus MoveBaseAction::onRunning()
{
  // Pretend that we are checking if the reply has been received
  // you don't want to block inside thisfunction too much time.
  std::this_thread::sleep_for(chr::milliseconds(10));

  // Pretend that, after a certain amount of time,
  // we have completed the operation
  if(chr::system_clock::now() >= _completion_time)
  {
    std::cout << "[ MoveBase: FINISHED ]" << std::endl;
    return BT::NodeStatus::SUCCESS;
  }
  return BT::NodeStatus::RUNNING;
}

void MoveBaseAction::onHalted()
{
  printf("[ MoveBase: ABORTED ]");
}
```

### 序列 VS 反应序列

**以下示例应使用一个简单的 SequenceNode。**

```
 <root BTCPP_format="4">
     <BehaviorTree>
        <Sequence>
            <BatteryOK/>
            <SaySomething   message="mission started..." />
            <MoveBase           goal="1;2;3"/>
            <SaySomething   message="mission completed!" />
        </Sequence>
     </BehaviorTree>
 </root>
```

```
int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerSimpleCondition("BatteryOK", std::bind(CheckBattery));
  factory.registerNodeType<MoveBaseAction>("MoveBase");
  factory.registerNodeType<SaySomething>("SaySomething");

  auto tree = factory.createTreeFromText(xml_text);

  // Here, instead of tree.tickWhileRunning(),
  // we prefer our own loop.
  std::cout << "--- ticking\n";
  auto status = tree.tickOnce();
  std::cout << "--- status: " << toStr(status) << "\n\n";

  while(status == NodeStatus::RUNNING)
  {
    // Sleep to avoid busy loops.
    // do NOT use other sleep functions!
    // Small sleep time is OK, here we use a large one only to
    // have less messages on the console.
    tree.sleep(std::chrono::milliseconds(100));

    std::cout << "--- ticking\n";
    status = tree.tickOnce();
    std::cout << "--- status: " << toStr(status) << "\n\n";
  }

  return 0;
}
```

**Expected output:**

```
--- ticking
[ Battery: OK ]
Robot says: mission started...
[ MoveBase: SEND REQUEST ]. goal: x=1.0 y=2.0 theta=3.0
--- status: RUNNING

--- ticking
--- status: RUNNING

--- ticking
[ MoveBase: FINISHED ]
Robot says: mission completed!
--- status: SUCCESS
```

您可能已经注意到，在调用 executeTick() 时，MoveBase 在第一次和第二次返回 RUNNING，最终在第三次返回 SUCCESS。

`BatteryOK` 只执行一次.

如果改用 `ReactiveSequence`，当子任务 `MoveBase` 返回`RUNNING` 时，该序列将重新启动，并再次执行条件 `BatteryOK`。

如果在任何时候`BatteryOK` 返回 FAILURE，`MoveBase` 操作将会被中断（确切地说，是停止）。

```
 <root>
     <BehaviorTree>
        <ReactiveSequence>
            <BatteryOK/>
            <Sequence>
                <SaySomething   message="mission started..." />
                <MoveBase           goal="1;2;3"/>
                <SaySomething   message="mission completed!" />
            </Sequence>
        </ReactiveSequence>
     </BehaviorTree>
 </root>
```

**预期输出：**

```
--- ticking
[ Battery: OK ]
Robot says: mission started...
[ MoveBase: SEND REQUEST ]. goal: x=1.0 y=2.0 theta=3.0
--- status: RUNNING

--- ticking
[ Battery: OK ]
--- status: RUNNING

--- ticking
[ Battery: OK ]
[ MoveBase: FINISHED ]
Robot says: mission completed!
--- status: SUCCESS
```

## 事件驱动的树？

**提示：**

我们特意使用 `tree.sleep()`命令而非`std::this_thread::sleep_for()`，是有原因的！！！

应优先使用 `Tree::sleep()` 方法，因为当“发生变化”时，树中的节点可以中断该方法。

当调用 TreeNode::emitStateChanged() 方法时，Tree::sleep() 将会被中断。

当调用`Tree::sleep()` 的时候 `TreeNode::emitStateChanged()` 会被中断。