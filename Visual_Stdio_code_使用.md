# Visual Stdio code 使用
--------------------------------

## 源代码管理

## 用户代码片段

## 语法高亮自定义

## 调试 C++ & 编写makefile

## Markdown 的使用

- 二级列表：`Tab` 键 + 空格
- 插入目录：直接在文件中输入 `[TOC]` 然后按回车即可。

### 推荐插件

1. **One Dark Pro**
    * 插件作用：VS code 主题，配色漂亮且护眼。 
2. **Markdown Preview Enhanced**
    * 插件作用：
        - 增强预览界面：可在编辑区右击鼠标点击 Open Preview to the Side 即可开启预览，在预览界面按 `Esc` 键还可显示文件的目录。
        - 支持 Markdown 扩展语法：GFM、数学公式、图表、目录等
        - 支持插入目录：直接在文件中输入 `[TOC]` 然后按回车即可。
        - 支持引用文件：输入 `@import "文件名"` 即可，支持引用 ”.md、.csv、.jpg、.png、.gif、.pdf、.html“ 等格式。
        - 幻灯片制作
        - 导出文件：需要安装相应工具，例如 pdf，需要安装 princexml，配置其bin路径为系统环境变量，重启 vs code 即可。
3. 

## 常用快捷键
- 格式化文档：Ctrl + Shift + F


## C/C++ 调试问题
1. 可以编译成功，但是无法单步调试，因为 makefile 里面 g++ 没有添加 -g 参数，即使是编译中间文件(g++ -c)，也需要添加 -g 参数，示例：
   * window 下 makefile：
  
```c
CC = g++

PROJECT = dlt698

LOCALDIR = .
CFLAGS = -w -fPIC

INCLUDE = -I$(LOCALDIR)/include  \
          -I$(LOCALDIR)/../include

LIB = -lpthread -L$(LOCALDIR)/../lib 
OBJ = main.o	dlt698.o

target : $(PROJECT)

$(PROJECT) : $(OBJ)
	$(CC) -g -o $@   $(INCLUDE) $^  $(LIB)
#	-cp $@ $(LOCALDIR)/../target

main.o : main.cpp
	$(CC) $(CFLAGS) $< -c -g $(INCLUDE)
dlt698.o : dlt698.cpp
	$(CC) $(CFLAGS) $< -c -g $(INCLUDE)

.PHONY:clean
clean:
	del *.o
	del *.exe

```

   * Linux 下 makefile

```c
CC = g++

PROJECT = dlt698

LOCALDIR = .
CFLAGS = -w -fPIC

INCLUDE = -I$(LOCALDIR)/include  \
          -I$(LOCALDIR)/../include

LIB = -lpthread -L$(LOCALDIR)/../lib 
OBJ = main.o	dlt698.o

target : $(PROJECT)

$(PROJECT) : $(OBJ)
	$(CC) -g -o $@  $(INCLUDE) $^  $(LIB)
	-cp $@ $(LOCALDIR)/../target

main.o : main.cpp
	$(CC) $(CFLAGS) $< -c -g $(INCLUDE)
dlt698.o : dlt698.cpp
	$(CC) $(CFLAGS) $< -c -g $(INCLUDE)

.PHONY:clean
clean:
	-rm  $(PROJECT) -f
	-rm *.o  -f
	-rm $(LOCALDIR)/../target/* -f


```

2. 调试配置文件 lanunch.json ，里面的详细配置  
   * windows 下：
```json
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [
            {
                "name": "(g++) Launch",
                "type": "cppdbg",
                "request": "launch",
                "program": "${fileDirname}\\${fileBasenameNoExtension}.exe", //这里是填写你的带路径的可执行文件
                "args": [],
                "stopAtEntry": false,
                "cwd": "${workspaceFolder}",
                "environment": [],
                "externalConsole": false,
                "MIMode": "gdb",
                "miDebuggerPath": "D:\\codeblocks\\MinGW\\bin\\gdb.exe",
                "preLaunchTask": "g++",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ],
                
            },   
        ]
    }
```
   * Linux 下：

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/src/dlt698", //这里填写你的带路径的可执行文件
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

## VS code 中的变量

|变量名|作用|
|--|--|
|${workspaceRoot} |当前打开的文件夹的绝对路径+文件夹的名字|
|${workspaceRootFolderName}|当前打开的文件夹的名字|
|${file}|当前打开正在编辑的文件名，包括绝对路径，文件名，文件后缀名|
|${relativeFile} |从当前打开的文件夹到当前打开的文件的路径|
|${fileBasename}|当前打开的文件名+后缀名，不包括路径|
|{fileBasenameNoExtension}|当前打开的文件的文件名，不包括路径和后缀名|
|${fileDirname} |当前打开的文件所在的绝对路径，不包括文件名|
|${fileExtname}|当前打开的文件的后缀名|
|${lineNumber}|当前打开的文件，光标所在的行数|

