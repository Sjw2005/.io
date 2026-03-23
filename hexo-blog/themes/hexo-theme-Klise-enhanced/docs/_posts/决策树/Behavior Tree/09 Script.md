# 09. Script

## 脚本语言简介

更多详细说明请参阅 [Introduction to Scripting](https://www.behaviortree.dev/docs/guides/scripting)。本教程提供了一个非常基础的示例，您可以将其作为初学时的练习平台。

## 脚本节点和先决条件节点

在我们的脚本语言中，变量是黑板中的条目。

在此示例中，我们使用 **Script** 节点来设置这些变量，并观察如何在 **SaySomething** 中将其作为输入端口进行访问。

支持的类型包括数字（整数和实数）、字符串以及已注册的 ENUM。

请注意，我们使用的 magic_enums 存在一些已知的限制。

其中一个显著的限制是，除非按照上述链接中的说明进行更改，否则其默认范围为 [-128, 128]。

我们将使用以下 XML：

```
<root BTCPP_format="4">
  <BehaviorTree>
    <Sequence>
      <Script code=" msg:='hello world' " />
      <Script code=" A:=THE_ANSWER; B:=3.14; color:=RED " />
      <Precondition if="A>B && color != BLUE" else="FAILURE">
        <Sequence>
          <SaySomething message="{A}"/>
          <SaySomething message="{B}"/>
          <SaySomething message="{msg}"/>
          <SaySomething message="{color}"/>
        </Sequence>
      </Precondition>
    </Sequence>
  </BehaviorTree>
</root>
```

**我们预计以下黑板条目应包含：**

- **msg**：字符串“hello world”
- **A**：与别名 THE_ANSWER 对应的整数值。
- **B**：实数值 3.14
- **C**：与枚举 RED 对应的整数值。

因此，预期输出为：

```
Robot says: 42.000000
Robot says: 3.140000
Robot says: hello world
Robot says: 1.000000
```

**The C++ code is:**

```
enum Color
{
  RED = 1,
  BLUE = 2,
  GREEN = 3
};

int main()
{
  BehaviorTreeFactory factory;
  factory.registerNodeType<DummyNodes::SaySomething>("SaySomething");

  // We can add these enums to the scripting language.
  // Check the limits of magic_enum
  factory.registerScriptingEnums<Color>();

  // Or we can manually assign a number to the label "THE_ANSWER".
  // This is not affected by any range limitation
  factory.registerScriptingEnum("THE_ANSWER", 42);

  auto tree = factory.createTreeFromText(xml_text);
  tree.tickWhileRunning();

  return 0;
}
```