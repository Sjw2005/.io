# 10. Loggers and Observer

## 日志记录器接口:

BT.CPP 提供了一种在运行时向树中添加**日志记录器**的方法，通常是在树创建完成之后、开始处理事件之前。

“日志记录器”是一种类，每当 TreeNode 更改状态时，该类的回调函数就会被调用；它是所谓[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)的一种非侵入式实现。

更具体地说，将被调用的回调函数是：

```
  virtual void callback(
    BT::Duration timestamp, // When the transition happened
    const TreeNode& node,   // the node that changed its status
    NodeStatus prev_status, // the previous status
    NodeStatus status);     // the new status
```

## TreeObserver 类:

有时，特别是在实现 **单元测试** 时，了解某个节点返回 SUCCESS 或 FAILURE 的次数会非常方便。

例如，我们希望验证在某些条件下，某条分支会被执行，而另一条分支则不会被执行。

`TreeObserver` 是一个简单的日志记录器实现，它会为树中的每个节点收集以下统计信息：

```
struct NodeStatistics
  {
    // Last valid result, either SUCCESS or FAILURE
    NodeStatus last_result;
    // Last status. Can be any status, including IDLE or SKIPPED
    NodeStatus current_status;
    // count status transitions, excluding transition to IDLE
    unsigned transitions_count;
    // count number of transitions to SUCCESS
    unsigned success_count;
    // count number of transitions to FAILURE
    unsigned failure_count;
    // count number of transitions to SKIPPED
    unsigned skip_count;
    // timestamp of the last transition
    Duration last_timestamp;
  };
```

## 如何唯一标识一个节点

由于观察器允许我们收集特定节点的统计信息，因此我们需要一种方法来唯一标识该节点：

**可以使用两种机制：**

`TreeNode::UID()` 这是一个与树的[深度优先遍历](https://en.wikipedia.org/wiki/Depth-first_search)对应的唯一编号。

`TreeNode::fullPath()` 旨在作为特定节点的唯一且易于人类阅读的标识符。

我们使用“path”这一术语，因为典型的字符串值可能如下所示：

first_subtree/nested_subtree/node_name

换句话说，该路径包含了节点在子树层次结构中的位置信息。

“node_name”要么是 XML 中分配的 name 属性，要么是通过节点注册自动分配的，格式为节点注册名后跟“::”及 UID。

## 示例（XML）

请看以下 XML，就子树而言，它具有非平凡的层次结构：

```
<root BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
     <Fallback>
       <AlwaysFailure name="failing_action"/>
       <SubTree ID="SubTreeA" name="mysub"/>
     </Fallback>
     <AlwaysSuccess name="last_action"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="SubTreeA">
    <Sequence>
      <AlwaysSuccess name="action_subA"/>
      <SubTree ID="SubTreeB" name="sub_nested"/>
      <SubTree ID="SubTreeB" />
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="SubTreeB">
    <AlwaysSuccess name="action_subB"/>
  </BehaviorTree>
</root>
```

您可能会注意到，有些节点具有“name”XML属性，而有些则没有。

相应的**UID** -> **fullPath**对列表如下：

```
1 -> Sequence::1
2 -> Fallback::2
3 -> failing_action
4 -> mysub
5 -> mysub/Sequence::5
6 -> mysub/action_subA
7 -> mysub/sub_nested
8 -> mysub/sub_nested/action_subB
9 -> mysub/SubTreeB::9
10 -> mysub/SubTreeB::9/action_subB
11 -> last_action
```

## 示例（C++）

以下应用程序将：

1. 递归地打印树的结构。
2. 将`TreeObserver`附加到树上。
3. 打印`UID / fullPath`对。
4. 收集名为“last_action”的特定节点的统计信息。
5. 显示观察器收集的所有统计信息。

```
int main()
{
  BT::BehaviorTreeFactory factory;

  factory.registerBehaviorTreeFromText(xml_text);
  auto tree = factory.createTree("MainTree");

  // Helper function to print the tree.
  BT::printTreeRecursively(tree.rootNode());

  // The purpose of the observer is to save some statistics about the number of times
  // a certain node returns SUCCESS or FAILURE.
  // This is particularly useful to create unit tests and to check if
  // a certain set of transitions happened as expected
  BT::TreeObserver observer(tree);

  // Print the unique ID and the corresponding human readable path
  // Path is also expected to be unique.
  std::map<uint16_t, std::string> ordered_UID_to_path;
  for(const auto& [name, uid]: observer.pathToUID()) {
    ordered_UID_to_path[uid] = name;
  }

  for(const auto& [uid, name]: ordered_UID_to_path) {
    std::cout << uid << " -> " << name << std::endl;
  }

  tree.tickWhileRunning();

  // You can access a specific statistic, using is full path or the UID
  const auto& last_action_stats = observer.getStatistics("last_action");
  assert(last_action_stats.transitions_count > 0);

  std::cout << "----------------" << std::endl;
  // print all the statistics
  for(const auto& [uid, name]: ordered_UID_to_path) {
    const auto& stats = observer.getStatistics(uid);

    std::cout << "[" << name
              << "] \tT/S/F:  " << stats.transitions_count
              << "/" << stats.success_count
              << "/" << stats.failure_count
              << std::endl;
  }

  return 0;
}
```