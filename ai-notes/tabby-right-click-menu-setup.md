# Tabby 终端右键菜单配置指南

## 需求背景

在 Windows 环境下，通过右键菜单直接打开 Tabby 终端并定位到当前目录。

**Tabby 路径**: `D:\Users\29580\AppData\Local\Programs\Tabby\Tabby.exe`

## 解决方案

### 创建注册表文件

通过修改 Windows 注册表添加右键菜单项。需要创建两个 `.reg` 文件：

#### 1. 添加右键菜单: `add_tabby_context_menu.reg`

```reg
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\shell\Tabby]
@="在此处打开 Tabby"
"Icon"="D:\\Users\\29580\\AppData\\Local\\Programs\\Tabby\\Tabby.exe,0"

[HKEY_CLASSES_ROOT\Directory\shell\Tabby\command]
@="\"D:\\Users\\29580\\AppData\\Local\\Programs\\Tabby\\Tabby.exe\" open \"%V\""

[HKEY_CLASSES_ROOT\Directory\Background\shell\Tabby]
@="在此处打开 Tabby"
"Icon"="D:\\Users\\29580\\AppData\\Local\\Programs\\Tabby\\Tabby.exe,0"

[HKEY_CLASSES_ROOT\Directory\Background\shell\Tabby\command]
@="\"D:\\Users\\29580\\AppData\\Local\\Programs\\Tabby\\Tabby.exe\" open \"%V\""
```

#### 2. 移除右键菜单: `remove_tabby_context_menu.reg`

```reg
Windows Registry Editor Version 5.00

[-HKEY_CLASSES_ROOT\Directory\shell\Tabby]
[-HKEY_CLASSES_ROOT\Directory\Background\shell\Tabby]
```

### 注册表项说明

| 注册表项 | 作用 |
|---------|------|
| `Directory\shell\Tabby` | 在文件夹上右键显示 |
| `Directory\Background\shell\Tabby` | 在文件夹空白处右键显示 |
| `Icon` | 显示 Tabby 图标 |
| `command` | 点击后执行的命令 |

## 遇到的问题与解决过程

### 问题 1: 中文显示乱码

**现象**: 右键菜单中的中文显示为乱码字符。

**原因**: 注册表文件默认需要 **GBK 编码**，而不是 UTF-8。

**解决**: 使用 GBK 编码保存 `.reg` 文件

```python
# Python 创建 GBK 编码文件
with open('add_tabby_context_menu.reg', 'w', encoding='gbk') as f:
    f.write(content)
```

### 问题 2: 图标不显示

**现象**: 右键菜单只显示文字，没有 Tabby 图标。

**原因**: Icon 路径格式错误。

### 问题 3: 点击后报错"该文件没有与之关联的应用"

**现象**: 点击右键菜单项后，弹出错误提示无法打开。

**原因**: `.reg` 文件中的反斜杠转义错误。

**核心问题**: 在 Windows `.reg` 文件中，路径的反斜杠必须写成 **双反斜杠 `\\`**

| 错误格式 | 正确格式 |
|---------|---------|
| `D:\Users\29580\...` | `D:\\Users\\29580\\...` |

注册表编辑器读取时，会将 `\\` 解析为单个 `\`。

## Tabby 命令行参数

通过 `Tabby.exe --help` 查看可用参数：

```
命令：
  Tabby.exe open [directory]    open a shell in a directory
  Tabby.exe run [command...]    run a command in the terminal
  Tabby.exe profile [name]      open a tab with specified profile
  ...
```

**打开指定目录的正确命令**: `Tabby.exe open "目录路径"`

## 最终正确的注册表格式

```reg
"Icon"="D:\\Users\\29580\\AppData\\Local\\Programs\\Tabby\\Tabby.exe,0"
```

关键点：
- 路径反斜杠双写: `D:\\Users` (不是 `D:\Users`)
- 图标索引: `,0` 表示使用第一个图标
- 命令格式: `open "%V"` (`%V` 是选中目录的变量)

## 使用方法

1. **安装右键菜单**: 双击 `add_tabby_context_menu.reg`
2. **确认提示**: 点击"是"确认修改注册表
3. **测试**: 右键任意文件夹，选择"在此处打开 Tabby"
4. **卸载**: 双击 `remove_tabby_context_menu.reg`

## 文件存放位置

建议将注册表文件放在项目目录中便于管理：

```
uv_python/
├── add_tabby_context_menu.reg
├── remove_tabby_context_menu.reg
└── README.md (更新文档)
```

## 参考资源

- [Tabby GitHub](https://github.com/Eugeny/tabby)
- [Tabby 使用教程](https://m.blog.csdn.net/m0_73310000/article/details/142146595)

---

**创建日期**: 2026-02-21
**适用环境**: Windows 10/11, Tabby Terminal
