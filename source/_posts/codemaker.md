title: IDEA代码生成插件CodeMaker
date: 2017-02-03 20:11:34
tags: [idea-plugin]
categories: 编程实践
---

## 前言
Java 开发过程中经常会遇到编写重复代码的事情，例如说：编写领域类和持久类的时候，大部分时候它们的变量名称，类型是一样的，在编写领域类的时候常常要重复写类似的代码。类似的问题太多，却没找到可以支持自定义代码模板的插件，只能自己动手，丰衣足食，开发了一个 IDEA 的代码生成插件，通过 Velocity 支持自定义代码模板来生成代码。

{% codemaker.gif %}

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
在 Java 类编辑界面右键“Generate”，选择对应模板即可自动生成代码到当前类的包，大部分情况下生成的代码已经解决了百分之八十的问题，只需稍作修改，移动到合适的包中，就能快速完成代码编写。
{% asset_img codemaker0.png %}

如果代码模板需要除了当前类之外的类作为上下文，可以通过类选择框进行选择。
{% asset_img codemaker1.png %}

目前自带的两个模板：

1. **Model**：根据当前类生成一个与其拥有类似属性的类，用于自动生成持久类对应的领域类（在持久类拥有超过10个属性的情况下，能够节省大量时间）。
2. **Converter**：该模板需要两个类作为输入的上下文，用于自动生成领域类与持久类的转化类。

上面两个模板是我自己工作中常用的模板，仅供大家参考，自带的模板可能满足不了大家的需求，**所以插件支持自定义新的代码模板**。

## 模板配置
{% asset_img codemaker3.png %}
1. **增加模板**：点击“Add Template”后，填写相关配置（都不能为空），点击保存后即可生效，无需重启。（感谢`khotyn`提醒）
2. **删除模板**：点击“Delete Template”就能将该模板删除

{% asset_img codemaker2.png %}
1. **Template Name**：在生成菜单中显示的名称，英文命名
2. **Class Number**：该模板需要的输入上下文类的数量，例如：如果为 1，,将当前的类作为输入：`$class0`；如果为 2，需要用户再选择一个类作为输入：`$class0, $class1`。
3. **Class Name**：生成的类的名称，支持通过 Velocity 进行配置，上下文为跟代码模板的相同。

## 模板上下文
模板上下文包含了以下变量：

```
########################################################################################
##
## Common variables:
##  $YEAR - yyyy
##  $TIME - yyyy-MM-dd HH:mm:ss
##  $USER - user.name
##
## Available variables:
##  $class0 - the context class
##  $class1 - the selected class, like $class2, $class2
##  $ClassName - generate by the config of "Class Name", the generated class name
##
## Class Entry Structure:
##  $class0.className - the class Name
##  $class0.packageName - the packageName
##  $class0.importList - the list of imported classes name
##  $class0.fields - the list of the class fields
##          - type: the field type
##          - name: the field name
##          - modifier: the field modifier, like "private"
##  $class0.methods - the list of class methods
##          - name: the method name
##          - modifier: the method modifier, like "private static"
##          - returnType: the method returnType
##          - params: the method params, like "(String name)"
##
########################################################################################
```
具体用法可参考自带的代码模板，通过模板上下文提供的定制能力，可以让每个用户都定制自己的风格的代码模板。

  [1]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker.gif
  [2]: https://github.com/x-hansong/CodeMaker
  [3]: https://github.com/x-hansong/CodeMaker/releases/download/1.0/CodeMaker.zip
  [4]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker0.png
  [5]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker1.png
  [6]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker3.png
  [7]: http://7xjtfr.com1.z0.glb.clouddn.com/codemaker2.png