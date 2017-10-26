---
title: Unity中画实线与虚线
date: 2016-11-24
tags:
- 画线
- Unity
categories: UnityScript
---

以前用过Vectrosity来画过线，但时间久了忘记怎么用了，也忘记能不能画虚线了。试了一下Unity的LineRenderer加上一个材质来画虚线，但是它是把我们的贴图给拉伸覆盖创建出来的mesh的，忘记保存我的实验效果了。。。可能改改Shader还可以用吧。针扎一番后决定自己用GL写，由于Unity中GL与真正的GL有差距，所以虚线费了点事。

### 实现方式
``` csharp

using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public enum LineType
{
    Dashed,
    Line
}

public class VectorLines : MonoBehaviour
{
    private static VectorLines inst;
    public static VectorLines instance
    {
        get
        {
            if (inst == null)
            {
                inst = Camera.main.transform.gameObject.AddComponent<VectorLines>();
            }
            return inst;
        }
    }
    private Dictionary<Line, Callback0> lines;
    private Material lineMaterial;

    void Awake()
    {
        lines = new Dictionary<Line, Callback0>();
        lineMaterial = new Material("Shader \"Lines/Colored Blended\" {" +
                "SubShader { Pass {" +
            "   BindChannels { Bind \"Color\",color }" +
            "   Blend SrcAlpha OneMinusSrcAlpha" +
            "   ZWrite Off Cull Off Fog { Mode Off }" +
            "} } }");

        lineMaterial.hideFlags = HideFlags.HideAndDontSave;

        lineMaterial.shader.hideFlags = HideFlags.HideAndDontSave;

    }

    public void Draw(Line line, Callback0 draw)
    {
        if (lines.ContainsKey(line)) 
            lines.Remove(line);
        lines.Add(line, draw);
    }

    public void Cancel(Line line)
    {
        if (lines.ContainsKey(line))
            lines.Remove(line);
    }

    void OnPostRender()
    {
        if (lines.Count > 0)
        {
            lineMaterial.SetPass(0);
            GL.Begin(GL.LINES);
            GL.Color(Color.gray);
            foreach (var line in lines.Values)
            {
                line();
            }
            GL.End();
        }
    }
}

public class Line
{
    private LineType lineType;
    private Vector3[] vectors;
    private float length;
    private Vector3 dir;
    private Vector3 next;
    private Vector3 cur;

    public Line()
    {
        lineType = LineType.Line;
        vectors = new []{Vector3.zero, Vector3.left};
        length = 1;
        ReDraw();
    }
    public Line(LineType lt, Vector3[] vecs, float len = 0.2f)
    {
        lineType = lt;
        vectors = vecs;
        length = len;
        ReDraw();
    }

    public void SetType(LineType lt)
    {
        if(lineType == lt) return;
        lineType = lt;
        ReDraw();
    }

    public void SetVectors(Vector3[] vecs)
    {
        if(vectors == vecs) return;
        vectors = vecs;
        ReDraw();
    }

    /// <summary>
    /// Can use only in dashed mode
    /// </summary>
    public void SetLength(float len)
    {
        if(length == len) return;
        length = len;
        ReDraw();
    }

    public void Cancel()
    {
        VectorLines.instance.Cancel(this);
    }

    void ReDraw()
    {
        if(vectors.Length <= 1) {Debug.LogError("vectors' length canot less than 2!");return;}
        if (lineType == LineType.Line)
        {
            VectorLines.instance.Draw(this, () =>
            {
                for (int i = 0; i < vectors.Length -1; i++)
                {
                    GL.Vertex(vectors[i]);
                    GL.Vertex(vectors[i+1]);
                }
            });
        }
        else if(lineType == LineType.Dashed)
        {
            if (length <= 0) { Debug.LogError("Length canot less than 0 in dashed mode!"); return;}
            VectorLines.instance.Draw(this, () =>
            {
                for (int i = 0; i < vectors.Length - 1; i++)
                {
                    cur = vectors[i];
                    dir = (vectors[i + 1] - vectors[i]).normalized;
                    GL.Vertex(cur);
                    next = cur + dir*length;
                    while (Vector3.Distance(next, vectors[i]) < Vector3.Distance(vectors[i + 1], vectors[i]))
                    {
                        GL.Vertex(next);
                        cur = next;
                        next = cur + dir*length;
                    }
                    GL.Vertex(vectors[i+1]);
                }
            });
        }
    }
}

```

### 使用方法
``` csharp
new Line(LineType.Dashed, v, 0.2f);//画虚线  
new Line();//画实线
```

### 具体效果
![使用效果](http://img.blog.csdn.net/20161124150745143?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

该做法优点是画多条线只增加一个DC,缺点也很明显，颜色单一，线条宽度。。。。试试其他有没有更好的方法吧。

### bug修正
当画虚线时长度分割正好为单数的时候会和下一条线连在一起，做出修改：
``` csharp
		if (length <= 0) { 
			Debug.LogError("Length canot less than 0 in dashed mode!"); 
			return;
		}  
		VectorLines.instance.Draw(this, () =>  
		{  
		    for (int i = 0; i < vectors.Length - 1; i++)  
		    {  
		        cur = vectors[i];  
		        dir = (vectors[i + 1] - vectors[i]).normalized;  
		        GL.Vertex(cur);  
		        flag = 1;  
		        next = cur + dir*length;  
		        while (Vector3.Distance(next, vectors[i]) < Vector3.Distance(vectors[i + 1], vectors[i]))  
		        {  
		            GL.Vertex(next);  
		            flag++;  
		            cur = next;  
		            next = cur + dir*length;  
		        }  
		        GL.Vertex(vectors[i+1]);  
		        flag++;  
		        if(flag % 2 != 0)  
		            GL.Vertex(vectors[i + 1]);  
		    }  
		});
```

