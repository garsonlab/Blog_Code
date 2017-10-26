---
title: Unity Scene场景自定义坐标轴
date: 2017-4-27
tags:
- Unity
- 坐标轴
categories: UnityScript
---

多看看别人的代码是没有坏处的，即使学不了人家的大框架，偶尔拾起一些小东西也是可以的。
最近扒了一下DoTween(声明一下源码是自己反编译的，只为学习)，看见了如何在Scene场景中添加标注和坐标轴，具体做法是，在你的脚本Editor中，比如你重定义某个mono脚本的Inspector显示中，加入OnSceneGUI函数，使用Handles进行操作。

``` csharp
void OnSceneGUI()  
{  
    if (_target.nodes.Count > 0)  
    {  
        //allow path adjustment undo:  
        Undo.RecordObject(_target, "Adjust Path");  

        //path begin and end labels:  
        Handles.Label(_target.nodes[0], "'" + _target.name + "' Begin");  
        Handles.Label(_target.nodes[_target.nodes.Count - 1], "'" + _target.name + "' End");  

        //node handle display:  
        for (int i = 0; i < _target.nodes.Count; i++)  
        {  
            _target.nodes[i] = Handles.PositionHandle(_target.nodes[i], Quaternion.identity);  
            if (i != 0 || i != _target.nodes.Count - 1)  
                Handles.Label(_target.nodes[i], i.ToString());  
        }  
        if (GUI.changed)  
        {  
            EditorUtility.SetDirty(_target);  
        }  
    }  
      
}
```

代码很简单，只是记录方法而已，具体的效果：

![图片来源我的csdn](http://img.blog.csdn.net/20170427161523433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

给开始和结束点添加了一个label, 每个节点添加了一个坐标轴和一个序号。其中蓝色的线使用Gizmos画的，可自行度娘。