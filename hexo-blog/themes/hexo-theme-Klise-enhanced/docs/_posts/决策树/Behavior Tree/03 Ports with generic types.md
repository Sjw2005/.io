# 03. Ports with generic types

在之前的教程中，我们介绍了输入和输出端口，其中端口的类型是 `std::string`。

接下来，我们将演示如何将通用 C++ 类型赋值给端口。

## **解析字符串**

**BehaviorTree.CPP** 支持将字符串自动转换为常见类型，例如 `int`、`long`、`double`、`bool`、`NodeStatus`等。用户自定义类型同样可以轻松支持。

例如：

```
// We want to use this custom type
struct Position2D
{
  double x;
  double y;
};
```

为了让 XML 加载器能够根据字符串实例化一个 `Position2D`，我们需要提供 `BT::convertFromString<Position2D>(StringView)` 的模板特化。

`Position2D` 如何序列化为字符串由您决定；在此示例中，我们仅用 *分号* 分隔两个数字。

```
// Template specialization to converts a string to Position2D.
namespace BT
{
    template <> inline Position2D convertFromString(StringView str)
    {
        // We expect real numbers separated by semicolons
        auto parts = splitString(str, ';');
        if (parts.size() != 2)
        {
            throw RuntimeError("invalid input)");
        }
        else
        {
            Position2D output;
            output.x     = convertFromString<double>(parts[0]);
            output.y     = convertFromString<double>(parts[1]);
            return output;
        }
    }
} // end namespace BT
```

- `StringView`是 [std::string_view](https://en.cppreference.com/w/cpp/header/string_view) 的 C++11 版本。您可以传入 `std::string` 或 `const char*`。
- 该库提供了一个简单的 `splitString` 函数。您也可以使用其他函数，例如 [boost::algorithm::split](https://www.boost.org/doc/libs/1_80_0/doc/html/boost/algorithm/split.html)。
- 我们可以使用特化函数 `convertFromString<double>()`。

## **示例**

如同上一节教程所示，我们可以创建两个自定义操作（Action），一个用于向端口写入数据，另一个用于从端口读取数据。

```
class CalculateGoal: public SyncActionNode
{
  public:
    CalculateGoal(const std::string& name, const NodeConfig& config):
      SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      return { OutputPort<Position2D>("goal") };
    }

    NodeStatus tick() override
    {
      Position2D mygoal = {1.1, 2.3};
      setOutput<Position2D>("goal", mygoal);
      return NodeStatus::SUCCESS;
    }
};

class PrintTarget: public SyncActionNode
{
  public:
    PrintTarget(const std::string& name, const NodeConfig& config):
        SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      // Optionally, a port can have a human readable description
      const char*  description = "Simply print the goal on console...";
      return { InputPort<Position2D>("target", description) };
    }

    NodeStatus tick() override
    {
      auto res = getInput<Position2D>("target");
      if( !res )
      {
        throw RuntimeError("error reading port [target]:", res.error());
      }
      Position2D target = res.value();
      printf("Target positions: [ %.1f, %.1f ]\n", target.x, target.y );
      return NodeStatus::SUCCESS;
    }
};
```

现在我们可以像往常一样连接输入/输出端口，使其指向黑板上的同一条目。

下一个示例中的树由 4 个操作组成：

- 使用操作 `CalculateGoal` 将 `Position2D` 的值存储到条目 **GoalPosition** 中。
- 调用`PrintTarget`。输入“target”将从黑板条目**GoalPosition**中读取。
- 使用内置操作`Script`将字符串“-1;3”赋值给键**OtherGoal**。字符串到`Position2D`的转换将自动完成。
- 再次调用PrintTarget。输入“target”将从条目OtherGoal中读取。

```
static const char* xml_text = R"(

 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root">
            <CalculateGoal goal="{GoalPosition}" />
            <PrintTarget   target="{GoalPosition}" />
            <Script        code=" OtherGoal:='-1;3' " />
            <PrintTarget   target="{OtherGoal}" />
        </Sequence>
     </BehaviorTree>
 </root>
 )";

int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<CalculateGoal>("CalculateGoal");
  factory.registerNodeType<PrintTarget>("PrintTarget");

  auto tree = factory.createTreeFromText(xml_text);
  tree.tickWhileRunning();

  return 0;
}
/* Expected output:

    Target positions: [1.1,2.3 ]
    Converting string:"-1;3"
    Target positions: [ -1.0,3.0 ]
*/
```