# 在sdformat中创建世界

从概念上讲，SDFormat 中的世界是一个环境，物理引擎可以在其中实例化和模拟模型。 世界使用
`<world>`标签创建。它可以包含各种元素，但是此文档仅涵盖`<model>`，因为我们描述如何创建由各种模型组成的模拟世界。完整的规格`<world>`可以在[这里](http://sdformat.org/spec?ver=1.4&elem=world).找到


> **注意**：关于`<world>`所需的名称（*name*）属性请参阅[附录](#appendix)

世界的一个最基本属性是它包含世界坐标系，该坐标系被定义为世界中所有动态刚体的标准惯性参考系。当模型作为 `<world>` 的直接子节点插入到世界中时，其姿态是相对于该坐标系表示的。有关设置模型姿态的更多信息，请参阅[指定姿态和模型运动学](/tutorials?tut=specify_pose)和[模型
运动学](/tutorial?tut=spec_model_kinematics)文档。


SDFormat中有两种方法可将模型插入世界。

## 内联定义的模型

第一种方法涉及直接在 `<world>` 标签内部定义模型。示例：

```xml
<?xml version="1.0" ?>
<sdf version="1.4">
  <world name="simple_world">
    <model name="ground">
      <link name="body">
        ...
      </link>
    </model>
    <model name="box">
      <pose>0 0 1 0 0 0</pose>
      <link name="body">
        ...
      </link>
    </model>
    <model name="sphere">
      <pose>10 0 2 0 0 0</pose>
      <link name="body">
        ...
      </link>
    </model>
  </world>
</sdf>
```
这是最简单的方法，因为它仅需要一个文件来描述世界。但是，它有一些缺点。

1. 如果需要多个相同模型但处于不同位置的实例，则必须复制 `<model> `标签的整个文本。
2. 在 world 文件中定义的模型不能在其他 `world` 或 `SDF` 文件中使用。

## 其他文件中定义的模型

为了减轻这些问题，sdformat v1.4引入了`<include>`在里面标记
`<world>`。使用这种方法，可以在单独的文件和
后来通过使用`<include>`标签。例子：

```xml
<!--ground/ground.sdf-->
<?xml version="1.0" ?>
<sdf version="1.4">
  <model name="ground">
    <link name="body">
      ...
    </link>
  </model>
</sdf>
```

```xml
<!--ground/model.config-->
<?xml version="1.0" ?>
<model>
  <name>ground</name>
  <sdf version="1.4">ground.sdf</sdf>
</model>
```

```xml
<!--box/box.sdf-->
<?xml version="1.0" ?>
<sdf version="1.4">
  <model name="box">
    <link name="body">
      ...
    </link>
  </model>
</sdf>
```

```xml
<!--box/model.config-->
<?xml version="1.0" ?>
<model>
  <name>box</name>
  <sdf version="1.4">box.sdf</sdf>
</model>
```

```xml
<!--sphere/sphere.sdf-->
<?xml version="1.0" ?>
<sdf version="1.4">
  <model name="sphere">
    <pose>1 2 3 0 0 0</pose>
    <link name="body">
      ...
    </link>
  </model>
</sdf>
```

```xml
<!--sphere/model.config-->
<?xml version="1.0" ?>
<model>
  <name>sphere</name>
  <sdf version="1.4">sphere.sdf</sdf>
</model>
```

```xml
<!--simple_world.sdf-->
<?xml version="1.0" ?>
<sdf version="1.4">
<world name="simple_world">
    <include>
      <uri>ground</uri>      
    </include>
    <include>
      <uri>box</uri>      
    </include>
    <include>
      <uri>sphere</uri>      
      <pose>10 0 2 0 0 0</pose>
    </include>
  </world>
</sdf>
```

如示例可以看出的模型`ground`, `box`， 和`sphere`是
在文件中定义`ground/ground.sdf`, `box/box.sdf`， 和
`sphere/sphere.sdf`分别与他们`model.config`文件。在
`simple_world.sdf`这`<include>`标签用于将模型包括在
世界。每个模型的姿势都可以被`<pose>`儿童标签
`<include>`。这在示例中证明了这一点
在模型的原始定义中是`1 2 3 0 0 0`但是被覆盖了
到`10 0 2 0 0 0`当插入世界时。由于模型的名称具有
独特，`<include>`还提供了覆盖名称的机制
随附的模型。因此，可以创建两个相同的实例
具有不同名称的模型，如下示例所示。

```xml
<?xml version="1.0" ?>
<sdf version="1.5">
<world name="simple_world_two_boxes">
    <include>
      <uri>box</uri>      
      <name>box1</name>
    </include>
    <include>
      <uri>box</uri>      
      <name>box2</name>
      <pose>4 0 1 0 0 0</pose>
    </include>
  </world>
</sdf>
```

> **注意**：功能上有限的版本`<include>`标签可用
> 在SDFORMAT v1.4中。此版本允许指定`<uri>`的
> 外部定义的模型，但不允许覆盖名称或姿势
> 插入的模型。

## 在地面上创建一个盒子和一个球

一个盒子的完全工作示例和放置在地面上的球体是
下面提供。请注意`<static>`在地面模型中使用标签
表明该模型不作为动态对象，应该是
仅考虑其碰撞和视觉特性。更多关于
`<static>`标签可以在[惯性
属性]（/tutorials？tut = spec_inertial）文档（即将推出）。

```xml
<?xml version="1.0" ?>
<sdf version="1.4">
  <world name="simple_world">
    <model name="ground">
      <static>true</static>
      <link name="ground_link">
        <collision name="collision1">
          <geometry>
            <plane>
              <normal>0 0 1</normal>
            </plane>
          </geometry>
        </collision>
        <visual name="visual1">
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="box">
      <pose>0 0 0.5 0 0 0</pose>
      <link name="body">
        <collision name="collision1">
          <geometry>
            <box>
              <size>1 1 1</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual1">
          <geometry>
            <box>
              <size>1 1 1</size>
            </box>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="sphere">
      <pose>10 0 1 0 0 0</pose>
      <link name="body">
        <collision name="collision1">
          <geometry>
            <sphere>
              <radius>1</radius>
            </sphere>
          </geometry>
        </collision>
        <visual name="visual1">
          <geometry>
            <sphere>
              <radius>1</radius>
            </sphere>
          </geometry>
        </visual>
      </link>
    </model>
  </world>
</sdf>
```

## 附录

这`<world>`标签有一个必需的`name`属性。它可以习惯
区分并行运行的多个世界。但是，这不是
一个非常常见的用例，将在本文中讨论。

世界元素未通过名称中的SDF文件中的其他位置提及
在其中指定`name`属性。相反，特殊名称`world`使用。
例如，将世界称为链接时`<joint>`元素，
特殊名称`world`只要没有兄弟姐妹链接，就指世界
有名字`world`.

```xml
<sdf version="1.4">
  <world name="world_with_joint">
    <model name="fixed_box">
      <link name="body"/>
      <joint name="j_fixed" type="fixed">
        <parent>world</parent> <!-- The name `world` is used instead of world_with_joint -->
        <child>body</child>
      </joint>
    </model>
  </world>
</sdf>
```
