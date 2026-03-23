# 07. multiple XML

## How to use multiple XML files

在我们之前介绍的示例中，我们总是从单个 XML 文件创建整个树及其子树。

但随着子树数量的增加，使用多个文件会更加方便。

## 子树

**subtree_A.xml：**

```
<root>
    <BehaviorTree ID="SubTreeA">
        <SaySomething message="Executing Sub_A" />
    </BehaviorTree>
</root>
```

**subtree_B.xml**:

```
<root>
    <BehaviorTree ID="SubTreeB">
        <SaySomething message="Executing Sub_B" />
    </BehaviorTree>
</root>
```

### Load multiple files manually (recommended)

我们来看一下名为 main_tree.xml 的文件，该文件应包含另外两个文件：

```
<root>
    <BehaviorTree ID="MainTree">
        <Sequence>
            <SaySomething message="starting MainTree" />
            <SubTree ID="SubTreeA" />
            <SubTree ID="SubTreeB" />
        </Sequence>
    </BehaviorTree>
</root>
```

要手动加载多个文件：

```
int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<DummyNodes::SaySomething>("SaySomething");

  // Find all the XML files in a folder and register all of them.
  // We will use std::filesystem::directory_iterator
  std::string search_directory = "./";

  using std::filesystem::directory_iterator;
  for (auto const& entry : directory_iterator(search_directory))
  {
    if( entry.path().extension() == ".xml")
    {
      factory.registerBehaviorTreeFromFile(entry.path().string());
    }
  }
  // This, in our specific case, would be equivalent to
  // factory.registerBehaviorTreeFromFile("./main_tree.xml");
  // factory.registerBehaviorTreeFromFile("./subtree_A.xml");
  // factory.registerBehaviorTreeFromFile("./subtree_B.xml");

  // You can create the MainTree and the subtrees will be added automatically.
  std::cout << "----- MainTree tick ----" << std::endl;
  auto main_tree = factory.createTree("MainTree");
  main_tree.tickWhileRunning();

  // ... or you can create only one of the subtrees
  std::cout << "----- SubA tick ----" << std::endl;
  auto subA_tree = factory.createTree("SubTreeA");
  subA_tree.tickWhileRunning();

  return 0;
}
/* Expected output:

Registered BehaviorTrees:
 - MainTree
 - SubTreeA
 - SubTreeB
----- MainTree tick ----
Robot says: starting MainTree
Robot says: Executing Sub_A
Robot says: Executing Sub_B
----- SubA tick ----
Robot says: Executing Sub_A
```

### 使用“include”添加多个文件

如果您希望将待包含的树信息移至 XML 文件本身，可以按照以下示例修改 main_tree.xml：

```
<root BTCPP_format="4">
    <include path="./subtree_A.xml" />
    <include path="./subtree_B.xml" />
    <BehaviorTree ID="MainTree">
        <Sequence>
            <SaySomething message="starting MainTree" />
            <SubTree ID="SubTreeA" />
            <SubTree ID="SubTreeB" />
        </Sequence>
    </BehaviorTree>
</root>
```

您可能已经注意到，我们在 main_tree.xml 中添加了两个相对路径，用于告知 `BehaviorTreeFactory` 如何查找所需的依赖项。

这些路径相对于 main_tree.xml 而言。

现在我们可以像往常一样创建树了：

```
factory.createTreeFromFile("main_tree.xml")
```