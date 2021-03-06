---
title: markdown使用指南
date: 2018-11-23 14:33:18
tags: 
    - markdown
categories:
    - markdown
---

Markdown是一种可以使用普通文本编辑器编写的标记语言，通过简单的标记语法，它可以使普通文本内容具有一定的格式。

## markdown使用指南
### 标题
标题能显示出文章的结构。行首插入1-6个 # ，每增加一个 # 表示更深入层次的内容，对应到标题的深度由 1-6 阶。

    H1 :# Header 1
    H2 :## Header 2
    H3 :### Header 3
    H4 :#### Header 4
    H5 :##### Header 5
    H6 :###### Header 6
<!-- more -->
### 文本样式
（带“*”星号的文本样式，在原版Markdown标准中不存在，但在其大部分衍生标准中被添加）

    链接 :[Title](URL)
    加粗 :**Bold**
    斜体字 :*Italics*
    *高亮 :==text==
    段落 : 段落之间空一行
    换行符 : 一行结束时输入两个空格
    列表 :* 添加星号成为一个新的列表项。
    引用 :> 引用内容
    内嵌代码 : `alert('Hello World');`
    画水平线 (HR) :--------
    方框：- [ ] -

### 图片
使用Markdown将图像插入文章，你需要在Markdown编辑器输入 **` ![]() `** 。 这时在预览面板中会自动创建一个图像上传框。你可以从电脑桌面拖放图片(.png, .gif, .jpg)到上传框, 或者点击图片上传框使用标准的图像上传方式。 如果你想通过链接插入网络上已经存在的图片，只要单击图片上传框的左下角的“链接”图标，这时就会呈现图像URL的输入框。想给图片添加一个标题, 你需要做的是将标题文本插图中的方括号， **`e.g;![This is a title]()`** .

### 脚注
脚注不存在于标准Markdown中。  
使用这样的占位符号可以将脚注添加到文本中:[^1]. 另外，你可以使用“n”而不是数字的[^n]所以你可以不必担心使用哪个号码。在您的文章的结尾，你可以如下图所示定义匹配的注脚，URL将变成链接:  
    ```
    这里是脚注[^1]
    [^1]: 这里是脚注的内容
    这里是脚注[^n]
    [^n]: 这里是脚注的内容
    ```

### 写代码
添加内嵌代码可以使用一对回勾号 **\`alert('Hello World')\`** .对于插入代码, Ghost支持标准的Markdown代码和GitHub Flavored Markdown (GFM) [4]  。标准Markdown基于缩进代码行或者4个空格位:
    ```

    \
        insert code
        insert code
        insert code
    \

    **删除\**
    ```

GFM 使用三个回勾号\`\`\`
    ```

    ``\`
    insert code
    insert code
    insert code
    ``\`

    **删除\**
    ```

### 例子
链接 
    ```
    This is a paragraph that contains a [link to example]()
    ```

列表格式 
    ``` 
    This paragraph contains a list of items.
    * Item 1
    * Item 2
    * Item three
    ```

使用Markdown 引用文本：	
    ```
    This paragraph has a quote
    
    > That is pulled out like this
    from the text my post.
    ```