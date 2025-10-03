# SDF快速入门指南

本指南将引导您了解 sdformat 的 [API](http://sdformat.org/api) , 向您展示如何使用它来解析您的SDF文件，您会发现该API非常规范而易于掌握。

## 前置条件

确保 [sdformat](/tutorials?tut=install) 已经安装到你的系统中.

## 解析模型SDF

### 开发代码

考虑下面的代码（C++ , 增加了部分注释）:

```c++
#include <iostream>

#include <sdf/sdf.hh> //-引入sdf头文件-
int main(int argc, const char* argv[])  //-主函数-
{
  // 参数检查
  if (argc < 2)
  {
    std::cerr << "使用: " << argv[0] 
              << " <sdf路径>" << std::endl;  //-输出信息：路径-
    return -1;
  }
  const std::string sdfPath(argv[1]);  //-定义SDF文件路径-

  //加载并检查SDF文件
  sdf::SDFPtr sdfElement(new sdf::SDF()); //-创建SDF对象-
  sdf::init(sdfElement);  //-初始化sdformat-
  if (!sdf::readFile(sdfPath, sdfElement)) // -读取SDF文件-
  {
    std::cerr << sdfPath << " 不是一个有效的SDF文件！" << std::endl; //-输出错误信息：非有效文件-
    return -2;
  }

  // 开始解析模型
  const sdf::ElementPtr rootElement = sdfElement->Root(); //-获取根元素-
  if (!rootElement->HasElement("model")) // -检查模型文件-
  {
    std::cerr << sdfPath << " 不是一个SDF模型文件！" << std::endl; //-输出错误信息：非模型文件-
    return -3;
  }
  const sdf::ElementPtr modelElement = rootElement->GetElement ("model"); //-获取模型元素-
  const std::string modelName = modelElement->Get<std::string>("name"); //-获取模型名称-
  std::cout << "发现" << modelName << " 模型!" << std::endl; //-输出信息：模型名称-

  // 解析模型链接
  sdf::ElementPtr linkElement = modelElement->GetElement("link"); //-获取模型链接-
  while (linkElement) // -循环遍历链接-
  {
    const std::string linkName = linkElement->Get<std::string>("name"); //-获取链接名称-
    std::cout << " 发现 " << linkName << " 链接在 " // -输出信息：链接名称-
              << modelName << " 模型中!" << std::endl;
    linkElement = linkElement->GetNextElement("link"); //-获取下一个模型链接-
  }

  // 解析模型接头
  sdf::ElementPtr jointElement = modelElement->GetElement("joint"); //-获取模型接头-
  while (jointElement) // -循环遍历接头-
  {
    const std::string jointName = jointElement->Get<std::string>("name"); //-获取接头名称-
    std::cout << "Found " << jointName << " joint in " // -输出信息：接头名称-
              << modelName << " model!" << std::endl; // -输出信息：模型名称-

    const sdf::ElementPtr parentElement = jointElement->GetElement("parent"); //-获取模型接头父级元素-
    const std::string parentLinkName = parentElement->Get<std::string>(); //  -获取链接父级名称-

    const sdf::ElementPtr childElement = jointElement->GetElement("child"); //-获取模型接头子级元素-
    const std::string childLinkName = childElement->Get<std::string>(); //-获取链接子级名称-

    std::cout << "接头" << jointName << " 承接 " << parentLinkName
              << "连接到" << childLinkName << " 链接link" << std::endl; //-输出信息：接头名称，链接父级名称，链接子级名称-

    jointElement = jointElement->GetNextElement("joint"); //-获取下一个模型接头-
  }

  return 0;
}
```

让我们来把它拆解一下.

----------------

```c++
sdf::SDFPtr sdfElement(new sdf::SDF());
sdf::init(sdfElement);
```

这是API的主要入口：任何SDF树，无论是在文件中还是字符串形式，都将被解析为这个对象。在调用```sdf::init()```时，将加载所有SDF版本的XML模式，以便稍后进行格式验证。


----------------

```c++
if (!sdf::readFile(sdfPath, sdfElement))
{
  std::cerr << sdfPath << " is not a valid SDF file!" << std::endl;
  return -2;
}
```

给定一个```sdf_path```，```sdf::readFile```将尝试将此文件解析为我们的```sdf_element``` 。如果它无法这样做，无论是由于文件无法访问，或者它不是一个有效的 SDF 或 URDF（是的，sdformat 可以无缝处理这两种格式！），它将返回```false```。

----------------

```c++
const sdf::ElementPtr rootElement = sdfElement->Root();
if (!rootElement->HasElement("model"))
{
  std::cerr << sdfPath << " is not a model SDF file!" << std::endl;
  return -3;
}
```

要遍历解析后的 SDF 树，我们从根节点开始。需要注意的是，这里的```root_element```并不是 SDF 中明确给出的元素（即典型世界 SDF 文件中的 ```<world>``` 元素），而是其上一级元素。这就是为什么我们要检查模型元素的存在。  

----------------
接下来，你会注意到一个遍历同一类型元素子元素的模式（也就是说，具有相同标签的元素）：
``` c++
sdf::ElementPtr linkElement = modelElement->GetElement("link");
while (linkElement){
  // 解析链接元素 
  linkElement = linkElement->GetNextElement("link");
}
```

一开始，我们从模型元素中获取第一个链接元素。如果不存在， `model_element->GetElement("link")` 将返回一个 `null` 指针，并且循环将被跳过。如果存在，我们将处理该链接元素。然后我们获取下一个链接元素，但这次使用 `link_element->GetNextElement("link")` 从当前链接元素获取，而不是从模型元素获取。后一种方法如果没有找到元素也会返回一个 `null` 指针，因此当没有更多该子元素时，我们最终会退出循环。


总而言之， `GetElement()` 用于检索子(*child*)元素，而 `GetNextElement()` 用于检索同级(*sibling*)元素。有了这个前提，你会发现对链接、关节、碰撞等的迭代会非常直观且易于理解。

现在让我们专注于关节元素的迭代：

```c++
const std::string jointName = jointElement->Get<std::string>("name");
std::cout << "Found " << jointName << " joint in "
          << modelName << " model!" << std::endl;
const sdf::ElementPtr parentElement = jointElement->GetElement("parent");
const std::string parentLinkName = parentElement->Get<std::string>();
const sdf::ElementPtr childElement = jointElement->GetElement("child");
const std::string childLinkName = childElement->Get<std::string>();
std::cout << "Joint " << jointName << " connects " << parentLinkName
          << " link to " << childLinkName << " link" << std::endl; 
```

正如你所猜到的，我们正在获取当前 `joint_element` 的 `name` 属性，然后检索其 `parent` 和 `child` 元素的值。值得注意的是 `Get()` 方法的所有使用场景： `Get()` 是一个模板方法，如果没有参数调用，它会返回给定的属性值或元素值本身（即标签之间的纯文本）。为此，它已被专门用于将此类值相应地解析为原始类型（即 `double` ）、常见 std 类型（即 `std::string` ）或 [`ignition::math`](http://gazebosim.org/libraries/math)  类型，用于最复杂的情况（如姿势）。

### 构建代码

创建一个空目录：

```sh
mkdir sdf_tutorial
cd sdf_tutorial
```


将代码复制到一个文件中并命名为 `check_sdf.cc` 。同时，添加一个 `CMakeLists.txt` 文件并将以下内容复制到其中：

```
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

find_package(SDFormat REQUIRED)
# find_package(sdformat<version of sdformat> REQUIRED) # sdformat7 or higher

include_directories(${SDFormat_INCLUDE_DIRS})
link_directories(${SDFormat_LIBRARY_DIRS})

add_executable(check_sdf check_sdf.cc)
target_link_libraries(check_sdf ${SDFormat_LIBRARIES}) 
```

**注意**：如果你使用的版本高于 6 ，你应该使用 `find_package(sdformat<version of sdformat> REQUIRED)` 来查找库

现在构建它：

```sh
mkdir build
cd build
cmake ..
make
```

你可以在[这里](http://models.gazebosim.org/)找到许多 SDF 示例进行操作。只需获取你选择的模型的 `model.sdf` ，并给它一个更有意义的名称。例如，在 `husky.sdf` 文件（网站上的 `husky/model.sdf` ）上运行 `./check_sdf` 将生成以下输出：

```
Found husky model!
Found base_link link in husky model!
Found back_left_wheel link in husky model!
Found back_right_wheel link in husky model!
Found front_left_wheel link in husky model!
Found front_right_wheel link in husky model!
Found back_left_joint joint in husky model!
Joint back_left_joint connects base_link link to back_left_wheel link
Found back_right_joint joint in husky model!
Joint back_right_joint connects base_link link to back_right_wheel link
Found front_left_joint joint in husky model!
Joint front_left_joint connects base_link link to front_left_wheel link
Found front_right_joint joint in husky model!
Joint front_right_joint connects base_link link to front_right_wheel link
```
