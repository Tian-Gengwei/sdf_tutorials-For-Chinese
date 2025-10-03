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

Next, you'll notice a pattern for iterating through an element children of the same type (that is, with the same tag):

```c++
sdf::ElementPtr linkElement = modelElement->GetElement("link");
while (linkElement)
{
    // parse link element
    linkElement = linkElement->GetNextElement("link");
}
```

At the beginning, we get the first link element from the model element.  If it doesn't exist, `model_element->GetElement("link")` will return a `null` pointer, and the loop will be skipped. If it does exist, we'll process that link element in the loop. We then get the next link element, but this time from the current link element instead of the model element using `link_element->GetNextElement("link")`. The latter method also returns a `null` pointer if no element could be found, so we'll eventually leave the loop when no more elements of that child remain.

To summarize, while `GetElement()` retrieves *child* elements, `GetNextElement()` retrieves *sibling* elements. With that in mind, you'll find the iterations over links, joints, collisions, etc., quite straightforward and easy to grasp.

----------------------
Let's now focus on the joints elements' iteration:

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

As you might guess, we're getting the `name` attribute of the current `joint_element` and then retrieving its `parent` and `child` elements' values. It is interesting to note all `Get()` method's use cases: `Get()` is a template method that returns the given attribute value or the element value itself (that is, the plain text in between tags) if called with no arguments. To this end, it has been specialized to parse such value accordingly into primitive types (i.e. `double`), common std types (i.e. `std::string`) or [`ignition::math`](http://gazebosim.org/libraries/math) types for the most complex ones (like poses).

### Building the code

Create an empty directory:

```sh
mkdir sdf_tutorial
cd sdf_tutorial
```

Copy the code into a file and name it `check_sdf.cc`. Along with it, add a `CMakeLists.txt` file and copy the following into it:

```
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

find_package(SDFormat REQUIRED)
# find_package(sdformat<version of sdformat> REQUIRED) # sdformat7 or higher

include_directories(${SDFormat_INCLUDE_DIRS})
link_directories(${SDFormat_LIBRARY_DIRS})

add_executable(check_sdf check_sdf.cc)
target_link_libraries(check_sdf ${SDFormat_LIBRARIES})
```

**Note**: If you are using a higher version than 6 you should use to find the library `find_package(sdformat<version of sdformat> REQUIRED)`

Now build it:

```sh
mkdir build
cd build
cmake ..
make
```

You can find plenty of SDF samples to play with [here](http://models.gazebosim.org/). Just fetch the `model.sdf` of your model of choice and give it a more meaningful name. For example, running `./check_sdf` on the `husky.sdf` file (`husky/model.sdf` on the site) will generate the following output:

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
