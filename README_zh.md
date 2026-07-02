[![构建状态](https://github.com/girishchandranc/autosarfactory/workflows/Build/badge.svg)](https://github.com/girishchandranc/autosarfactory/actions)

# Autosar 建模工具

AutosarFactory 提供了一套便捷的方法来读取、创建和修改符合 AUTOSAR 标准的 `.arxml` 文件。`autosarfactory` 文件夹包含基于最新 AUTOSAR 版本 schema 的实现。如需旧版本 AUTOSAR 的实现，请查看 `autosar_releases` 文件夹。

## 使用方式

- 克隆仓库。
- 手动安装包：

    ```bash
    $ python setup.py install
    ```

  > 若同时安装了 `python2` 和 `python3`，请使用 `python3`。

- 在 Python 脚本中导入 `autosarfactory` 包。
- 然后就可以愉快地进行 AUTOSAR 建模了。

### 读取文件

```python
# 读取文件列表
files = ['component.arxml', 'datatypes.arxml']
root, status = autosarfactory.read(files)

# 读取文件夹列表
folders = ['folder1', 'folder2']
root, status = autosarfactory.read(folders)
```

`read` 方法处理输入的文件/文件夹，并返回根节点（包含所有文件的合并信息）。若传入的是文件夹，该方法会递归搜索其中的 `.arxml` 文件并全部读入。`status` 返回值为布尔值，表示文件/文件夹是否读取成功。

### 创建新文件

```python
newPack = autosarfactory.new_file('newFile.arxml', defaultArPackage='NewPack')
```

创建新的 arxml 文件并使用指定的包名，返回 `ARPackage` 对象。

- 若未提供包名，默认包名为 `'RootPackage'`。
- 若目标文件已存在，该方法会抛出 `FileExistsError`。如需覆盖，请将参数 `overWrite` 设为 `True`。

### 访问属性与引用

模型元素提供 `get_<属性/引用>` 方法来获取已有的属性值和引用值。

> 对于多重引用，该方法返回一个值列表。

### 修改属性与引用

所有元素都提供 `set_<属性/引用>` 方法来修改属性值或引用值。

> 对于多重引用，还额外提供 `add_<引用>`、`remove_<引用>` 方法。

### 添加新的模型元素

所有父类都提供 `new_<元素>` 方法来创建子元素。

```python
rootPack = autosarfactory.new_file('newFile.arxml', defaultArPackage='RootPack')
newPack = rootPack.new_ARPackage('NewPack')

# 新建 Application 组件
asw1 = newPack.new_ApplicationSwComponentType('asw1')
asw1.new_PPortPrototype('outPort')

# 新建 SenderReceiver 接口
srIf = newPack.new_SenderReceiverInterface('srif1')
srIf.new_DataElement('de1')
```

### 通过路径访问元素

文件被工具读取后，即可通过路径访问元素。

```python
files = ['component.arxml', 'datatypes.arxml']
autosarfactory.read(files)
swc = autosarfactory.get_node('/Swcs/asw1')
uint8DataType = autosarfactory.get_node('/DataTypes/baseTypes/uint8')
```

### 保存选项

#### Save（增量保存）

工具提供 `save` 方法来保存对模型所做的更改。

```python
files = ['component.arxml', 'datatypes.arxml']
autosarfactory.read(files)

rootPack = autosarfactory.new_file('newFile.arxml', defaultArPackage='RootPack')
newPack = rootPack.new_ARPackage('NewPack')

# 新建 Application 组件
newcomp = newPack.new_ApplicationSwComponentType('newcomponent')
newcomp.new_PPortPrototype('outPort')

# 新建 SenderReceiver 接口
srIf = newPack.new_SenderReceiverInterface('srif1')
srIf.new_DataElement('de1')

# 保存更改
autosarfactory.save(['newFile.arxml'])
```

`save` 方法接受需要保存的文件列表。若不传入参数，则所有文件（输入的、新创建的）都会被保存。

#### SaveAs（合并另存）

工具提供 `saveAs` 方法将对模型的所有更改保存到单一的 arxml 文件中。

```python
files = ['component.arxml', 'datatypes.arxml']
autosarfactory.read(files)

rootPack = autosarfactory.new_file('newFile.arxml', defaultArPackage='RootPack')
newPack = rootPack.new_ARPackage('NewPack')

# 新建 Application 组件
newcomp = newPack.new_ApplicationSwComponentType('newcomponent')
newcomp.new_PPortPrototype('outPort')

# 新建 SenderReceiver 接口
srIf = newPack.new_SenderReceiverInterface('srif1')
srIf.new_DataElement('de1')

# 保存更改
autosarfactory.saveAs('mergedFile.arxml')
```

### 导出元素

工具提供 `export_to_file` 方法将特定元素及其 AR-Package 层级导出到文件。该功能仅支持 `CollectableElement`（即 AR-Package 以及所有 PackageableElement，包括 ApplicationSwComponentType、SR interface、Signals 等——简而言之，任何直接隶属于 AR-Package 的元素）。

- 方式一（使用节点自带的导出方法）：

```python
autosarfactory.read(['component.arxml'])
swc = autosarfactory.get_node('/Swcs/swc1')
swc.export_to_file('swc1Export.arxml', overWrite=True)
```

- 方式二（使用模块级导出函数，传入节点）：

```python
autosarfactory.read(['component.arxml'])
swc = autosarfactory.get_node('/Swcs/swc1')
autosarfactory.export_to_file(swc, 'swc1Export.arxml', overWrite=True)
```

### MCP 服务器

`mcp_server/` 文件夹包含一个 [MCP（模型上下文协议）](https://modelcontextprotocol.io) 服务器，可将 autosarfactory API 暴露给 Claude Code 等 AI 编程助手。AI 代理可以通过它查询完整的 API 参考、发现元素层级结构、查找枚举值并生成正确的 autosarfactory Python 脚本——全程无需手动阅读源码。

配置说明和完整细节请参见 [mcp_server/README.md](mcp_server/README.md)。

### Autosar 可视化工具

本包还包含一个 Autosar 模型的图形化可视化工具，只需将 autosar 根节点传入 `show_in_ui` 方法即可打开。例如：

```python
files = ['component.arxml', 'datatypes.arxml']
root, status = autosarfactory.read(files)
autosar_ui.show_in_ui(root)
```

以下是可视化工具的界面截图：

![AutosarVisualizer-2021-01-26 130700](https://user-images.githubusercontent.com/55708936/105837616-a1acdd00-5fd7-11eb-92ee-6255ae202749.jpg)

可视化工具主要由 4 个视图和 1 个菜单栏组成：

- **Autosar 浏览器** —— 以树状结构展示模型中的所有元素。
- **属性视图** —— 显示 Autosar 浏览器中当前选中元素的属性及对应值。
- **被引用者视图** —— 列出模型中引用了 Autosar 浏览器中当前选中元素的那些元素。
- **搜索视图** —— 提供对模型中任意元素的搜索功能。可通过视图顶部的下拉框选择搜索类型。共有 3 种搜索方式：按元素短名搜索、按正则表达式搜索、按 Autosar 类型搜索。

## 示例

请查看 `Examples` 文件夹中的脚本，其中创建了一个基础的通信矩阵。

更多用法请参考 `tests/test_autosarmodel.py`。
