# Windows 常用操作

## 查看dll占用

```cpp
Tasklist /m rogamelibs.dll
Tasklist /m msvcr100.dll
```

![Untitled](Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled.png)

## 查看端口占用

```powershell
netstat -ano | findstr 端口号   //查看这个端口情况
tasklist | findstr 进程PID      //查看对应的进程ID
taskkill -PID 进程PID -F        //杀死进程
```

## 将应用添加到右键

1、win+R，运行regedit

2、在HKEY_CLASSES_ROOT目录下，依次展开：Directory\Background\shell，如图所示

![Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%201.png](Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%201.png)

3、选中shell，右键，新建项（名字随意），再在项下新建一个名为command的项

4、编辑command下的项，将值设置为应用的启动路径

![Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%202.png](Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%202.png)

5、在shell下新建的那个项中的默认的之中，写入你希望右键显示的文字，还可以添加图标

![Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%203.png](Windows%20%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%20d4b2cf687b604ff2b19524c7122d4bff/Untitled%203.png)







