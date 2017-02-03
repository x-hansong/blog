title: IDEA代码生成插件CodeMaker
date: 2017-02-03 20:11:34
tags: [idea-plugin]
categories:
---

## 前言
Java 开发过程中经常会遇到手工编写重复代码的事情，例如说：编写领域类和持久类的时候，大部分时候它们的变量名称，类型是一样的，在编写领域类的时候常常要重复写类似的代码。所以开发了一个 IDEA 的代码生成插件，通过 Velocity 支持自定义代码模板来生成代码。

![demonstration][1]

**项目地址**：[CodeMaker][2]

## 主要功能
1. 支持增加自定义代码模板（Velocity）
2. 支持选择多个类作为代码模板的上下文

## 安装
下载插件：[CodeMaker.zip][3]

1. 打开设置，选择“Plugin”
2. 在右边的框中点击“Install plugin from disk”
3. 选择上面下载的“CodeMaker.zip”
4. 点击“Apply”，然后重启 IDEA。

## 使用
在 Java 类编辑界面右键“Generate”，选择对应模板即可自动生成代码到当前类的包，然后进行修改，并且移动到合适的位置。
![codemaker0][4]

如果代码模板需要除了当前类之外的类作为上下文，可以通过类选择框进行选择。
![codemaker1][5]

目前自带的两个模板：

1. **Model**：根据当前类生成一个与其拥有类似属性的类，用于自动生成持久类对应的领域类
2. **Converter**：该模板需要两个类作为输入的上下文，用于自动生成领域类与持久类的转化类。

上面两个模板是我自己工作中常用的模板，大家可以参考其写法，自己定义新的代码模板。

## 模板配置
![codemaker3][6]
1. **增加模板**：点击“Add Template”后，填写相关配置（都不能为空），点击保存后重启 IDEA 才能生效。
2. **删除模板**：点击“Delete Template”就能将该模板删除，同样需要重启才能生效。

![codemaker2][7]
1. **Template Name**：在生成菜单中显示的名称，英文命名
2. **Class Number**：该模板需要的输入上下文类的数量，例如：如果为 1，,将当前的类作为输入：`$class0`；如果为 2，需要用户再选择一个类作为输入：`$class0, $class1`。
3. **Class Name**：生成的类的名称，支持通过 Velocity 进行配置，上下文为跟代码模板的相同。


  [1]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker.gif
  [2]: https://github.com/x-hansong/CodeMaker
  [3]: https://github.com/x-hansong/CodeMaker/releases/download/1.0/CodeMaker.zip
  [4]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker0.png
  [5]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker1.png
  [6]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker3.png
  [7]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker2.png