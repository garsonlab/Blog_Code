---
title: unity点选构建Mesh并保存OBJ
date: 2016-6-8
tags:
- Unity
- Mesh
categories: Unity
---

最近有一份需求，就是让策划任意选择可一片区域，表明是有某种用途的。埋头写了两头，试了三四种方法，最终不得已用小方格来代替，并最终构建mesh保存下来，这样做程序的就很方便用了。我们的目标就是没有蛀牙大笑oh，应该是累死别人，轻松自己！！！

## 简单shader编写
写一个Shader的目的是来显示我们的所编辑的内容，具体代码如下：
``` cpp
Shader "Custom/BlockShader" {
	Properties {
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_Size ("Size",Vector) = (64,32,0,0) //在面板上显示长多少格，宽多少格
		_Color ("Color", Color) = (0,0,0,0.5)
		_RimColor ("Rim Color", Color) = (1,1,0,1)
		_RimPower ("Rim Power", Range(0,0.2)) = 0.01
		_HitPoint ("Hit Point",Vector) = (0.0,1,0.0,0.5)//鼠标点击点
		_SelectColor ("Select Color",Color) = (0,1,0,1)//选中颜色
		_HitColor ("Already hit Color", Color) = (0.2,0.8,0.9,0.5)
	}
	SubShader {
		Tags { "Queue"="Transparent" }
		
		CGPROGRAM
		#pragma surface surf Lambert alpha
		#pragma target 3.0

		sampler2D _MainTex;
		half4 _Size;
		fixed4 _Color;
		fixed4 _RimColor;
		float _RimPower;
		float4 _HitPoint;
		fixed4 _SelectColor;
		fixed4 _HitColor;

		struct Input {
			float2 uv_MainTex;
		};

		void surf (Input IN, inout SurfaceOutput o) {
			float2 uv = IN.uv_MainTex;
			fixed4 c = _Color;
			float radiox = 1.0 / (_Size.x * 2);//每格大小的一半
			float radioy = 1.0 / (_Size.y * 2);

			float nx = floor(uv.x/(radiox * 2)) * radiox * 2 + radiox;//找到格的中心点
			float ny = floor(uv.y/(radioy * 2)) * radioy * 2 + radioy;
			fixed4 tc = tex2D (_MainTex, float2(nx,ny) );
			if(tc.g > 0.85) //因为选中我设置为了绿色（0,1,0,1）
			{
				c = _HitColor;
				//if(uv.x >= nx-radiox && uv.x <nx+ radiox && uv.y >= ny-radioy && uv.y <= ny+radioy)
				//{ 
				//	c = _HitColor;
				//}
			}


			if(uv.x >= _HitPoint.x-radiox && uv.x <_HitPoint.x+ radiox && uv.y >= _HitPoint.y-radioy && uv.y <= _HitPoint.y+radioy)
			{ 
				c = _SelectColor;//选中颜色
			}

			if(fmod(uv.x ,(radiox * 2)) < (radiox * 2) * _RimPower)//边框
			{
				c = _RimColor;
			}
			if(fmod(uv.y ,(radioy * 2)) < (radioy * 2) * _RimPower)
			{
				c = _RimColor;
			}

			o.Albedo = c.rgb;
			o.Alpha = 0.5;
		}
		ENDCG
	} 
	FallBack "Diffuse"
}

```

我知道，以上写的很粗糙。。。但是，谁让我是pc上用的呢，谁还管效率是毛线
下一步，需要每套的脚本控制

## 编写控制脚本
主要是为了运行时，通过点击生成方格

``` csharp
using System.Collections.Generic;
using UnityEngine;

/**
 * 
 * 用于编辑器美术使用脚本，功能为画地形方块
 * 
 */
public class TerrianBlockControl : MonoBehaviour
{
    public int width;
    public int height;
    public Texture2D texture;
#if UNITY_EDITOR
    private Material material;
    private MeshRenderer meshRenderer;
    private RaycastHit hit;
    private float ratioW;
    private float ratioH;
    private int[,] matrix;

    void Start()
    {
        meshRenderer = GetComponent<MeshRenderer>();
        material = meshRenderer.material;
        texture = new Texture2D(width * 3, height * 3);

        material.SetTexture("_MainTex", texture);
        ratioW = 1.0f/width;
        ratioH = 1.0f/height;
        material.SetVector("_Size", new Vector4(width, height, 0, 0));
        matrix = new int[width,height];
        MatrixInit();
    }



    public void ChangeSize(int x, int y)
    {
        width = x;
        height = y;
        ratioW = 1.0f / width;
        ratioH = 1.0f / height;
        matrix = new int[width,height];
        MatrixInit();
    }

    public void Update()
    {
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        if (Physics.Raycast(ray, out hit))
        {//算出每个方格的正中心坐标
            float x = Mathf.FloorToInt(hit.textureCoord.x/ratioW)*ratioW + ratioW*0.5f;
            float y = Mathf.FloorToInt(hit.textureCoord.y / ratioH) * ratioH + ratioH * 0.5f;
            material.SetVector("_HitPoint",new Vector4(x,y,0,0));
        }
        if (Input.GetMouseButtonDown(0))
        {//算出所在格子的坐标
            int x = Mathf.FloorToInt(hit.textureCoord.x/ratioW) * 3 + 1;
            int y = Mathf.FloorToInt(hit.textureCoord.y/ratioH) * 3 + 1;
            Color  c = texture.GetPixel(x, y);
            if (c.g >= 0.9f)
            {
                texture.SetPixel(x, y, Color.black);
                matrix[(x - 1) / 3, (y - 1) / 3] = 0;
            }
            else
            {
                texture.SetPixel(x, y, Color.green);
                matrix[(x - 1) / 3, (y - 1) / 3] = 1;
            }
            texture.Apply(true);
            material.SetTexture("_MainTex",texture);
        }
    }

    void MatrixInit()
    {
        for (int i = 0; i < width; i++)
        {
            for (int j = 0; j < height; j++)
            {
                matrix[i, j] = 0;
            }
        }
    }

    List<rect> rects;
    [ContextMenu("Save")]
    public GameObject Save()
    {
        rects = new List<rect>();
        for (int i = 0; i < width; i++) //合并，把我们已经画好的方格进行合并
        {
            for (int j = 0; j < height; j++)
            {
                if (matrix[i, j] == 1)
                {
                    rect r = new rect();
                    r.dl = new Vector3(i,j,0);
                    r.dr = new Vector3(i+1,j,0);
                    r.ul = new Vector3(i,j+1,0);
                    r.ur = new Vector3(i+1,j+1);
                    r.length = 1;
                    matrix[i, j] = 0;
                    

                    for (int k = i+1; k < width; k++) //在width方向上进行合并
                    {
                        if (matrix[k, j] == 1)
                        {
                            r.dr=new Vector3(k+1,j,0);
                            r.ur = new Vector3(k+1,j+1,0);
                            matrix[k, j] = 0;
                            r.length++;
                        }
                        else
                        {
                            break;
                        }
                    }

                    for (int k = i-1; k >= 0; k--)
                    {
                        if (matrix[k, j] == 1)
                        {
                            r.dl = new Vector3(k,j,0);
                            r.ul = new Vector3(k,j+1,0);
                            matrix[k, j] = 0;
                            r.length++;
                        }
                        else
                        {
                            break;
                        }
                    }
                    rects.Add(r);
                }
            }
        }
        
        for (int i = 0; i < rects.Count; i++) //把合并过的，相邻的且大小相等的再合并
        {
            if (!rects[i].used && rects[i].length != 0)
            {
                for (int j = 0; j < rects.Count; j++)
                {//没有合并过  不是自身  
                    if (!rects[j].used && i != j && rects[j].length != 0)
                    {
                        if (rects[i].length == rects[j].length)
                        {
                            if (rects[i].dl.x == rects[j].dl.x)
                            {
                                if (rects[i].ul.y - rects[j].ul.y == 1)
                                {
                                    rects[i].dl = rects[j].dl;
                                    rects[i].dr = rects[j].dr;
                                    
                                    rects[i].used = true;
                                    rects[j].length = 0;
                                }
                                if (rects[i].ul.y - rects[j].ul.y == -1)
                                {
                                    rects[i].ul = rects[j].ul;
                                    rects[i].ur = rects[j].ur;

                                    rects[i].used = true;
                                    rects[j].length = 0;
                                }
                            }
                        }
                    }
                }
            }

            if (rects[i].length != 0 && rects[i].ul.y - rects[i].dl.y == 1)
                rects[i].used = true;
        }

        return creat(rects);
    }

    class rect
    {
        public Vector3 dl;
        public Vector3 dr;
        public Vector3 ul;
        public Vector3 ur;
        public int length;
        public bool used;
    }
//创建mesh
    GameObject creat(List<rect> rects_Old )
    {
        List<rect> rects = new List<rect>();
        foreach (var rr in rects_Old)
        {
            if(rr.used)
                rects.Add(rr);
        }
        GameObject mp = GameObject.Find("MaskParent");
        if(mp == null)
            mp = new GameObject("MaskParent");
        mp.transform.position = Vector3.zero;

        GameObject go = new GameObject(transform.name);
        go.transform.parent = mp.transform;
        go.transform.localPosition = Vector3.zero;
        MeshFilter mf = go.AddComponent<MeshFilter>();
        MeshRenderer mr = go.AddComponent<MeshRenderer>();
        mr.material = new Material(Shader.Find("Diffuse"));

        Mesh mesh = new Mesh();
        Vector3[] vectices = new Vector3[rects.Count*4];
        int[] trangles = new int[rects.Count * 6];

        Vector3 startPos = transform.position;

        float scalex = transform.localScale.x;
        float scaley = transform.localScale.y;
        startPos.x -= scalex * 0.5f;
        startPos.z -= scaley * 0.5f;

        int len = 0;
        for (int i = 0; i < rects.Count; i++)
        {//添加mesh的Vectices
            rect r = rects[i];
            Vector3 dl = new Vector3(startPos.x + ratioW * r.dl.x * scalex, startPos.z + ratioH * r.dl.y * scaley, 0);

            Vector3 dr = new Vector3(startPos.x + ratioW * r.dr.x * scalex, startPos.z + ratioH * r.dr.y * scaley, 0);

            Vector3 ul = new Vector3(startPos.x + ratioW * r.ul.x * scalex, startPos.z + ratioH * r.ul.y * scaley, 0);

            Vector3 ur = new Vector3(startPos.x + ratioW * r.ur.x * scalex, startPos.z + ratioH * r.ur.y * scaley, 0);


            vectices[i*4] = dl;
            vectices[i * 4 + 1] = dr;
            vectices[i * 4 + 2] = ul;
            vectices[i * 4 + 3] = ur;//根据上面的点顺时针画三角形
            trangles[i*6] = i*4;
            trangles[i * 6 + 1] = i * 4 +2;
            trangles[i * 6 + 2] = i * 4 +1;
            trangles[i * 6 + 3] = i * 4 +1;
            trangles[i * 6 + 4] = i * 4 +2;
            trangles[i * 6 + 5] = i * 4 +3;
        }

        mesh.vertices = vectices;
        mesh.triangles = trangles;
        mesh.RecalculateNormals();
        mesh.RecalculateBounds();
        mf.mesh = mesh;

        return go;
    }

#endif
}

```

根据以上两步能得到画好的mesh图形，还需要保存。

## 保存自创Mesh

这个很蛋疼，因为在运行状态下，原以为保存成预制没事，然后停止后还是尘归尘土归土。。。最后找了很久，感谢[这位的博客](http://blog.csdn.net/awnuxcvbn/article/details/50737192)帮助，成功导出成OBJ文件，在unity中生成FBX.

具体代码如下：
``` csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;
using UnityEngine;
public class ObjExporter
{
    //
    // Static Methods
    //
    public static void MeshToFile(MeshFilter mf, string filename, float scale)
    {
        using (StreamWriter streamWriter = new StreamWriter(filename))
        {
            streamWriter.Write(ObjExporter.MeshToString(mf, scale));
        }
    }

    public static string MeshToString(MeshFilter mf, float scale)
    {
        Mesh mesh = mf.mesh;
        Material[] sharedMaterials = mf.GetComponent<Renderer>().sharedMaterials;
        Vector2 textureOffset = mf.GetComponent<Renderer>().material.GetTextureOffset("_MainTex");
        Vector2 textureScale = mf.GetComponent<Renderer>().material.GetTextureScale("_MainTex");
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.Append("mtllib design.mtl").Append("\n");
        stringBuilder.Append("g ").Append(mf.name).Append("\n");
        Vector3[] vertices = mesh.vertices;
        for (int i = 0; i < vertices.Length; i++)
        {
            Vector3 vector = vertices[i];
            stringBuilder.Append(string.Format("v {0} {1} {2}\n", vector.x * scale, vector.z * scale, vector.y * scale));
        }
        stringBuilder.Append("\n");
        Dictionary<int, int> dictionary = new Dictionary<int, int>();
        if (mesh.subMeshCount > 1)
        {
            int[] triangles = mesh.GetTriangles(1);
            for (int j = 0; j < triangles.Length; j += 3)
            {
                if (!dictionary.ContainsKey(triangles[j]))
                {
                    dictionary.Add(triangles[j], 1);
                }
                if (!dictionary.ContainsKey(triangles[j + 1]))
                {
                    dictionary.Add(triangles[j + 1], 1);
                }
                if (!dictionary.ContainsKey(triangles[j + 2]))
                {
                    dictionary.Add(triangles[j + 2], 1);
                }
            }
        }
        for (int num = 0; num != mesh.uv.Length; num++)
        {
            Vector2 vector2 = Vector2.Scale(mesh.uv[num], textureScale) + textureOffset;
            if (dictionary.ContainsKey(num))
            {
                stringBuilder.Append(string.Format("vt {0} {1}\n", mesh.uv[num].x, mesh.uv[num].y));
            }
            else
            {
                stringBuilder.Append(string.Format("vt {0} {1}\n", vector2.x, vector2.y));
            }
        }
        for (int k = 0; k < mesh.subMeshCount; k++)
        {
            stringBuilder.Append("\n");
            if (k == 0)
            {
                stringBuilder.Append("usemtl ").Append("Material_design").Append("\n");
            }
            if (k == 1)
            {
                stringBuilder.Append("usemtl ").Append("Material_logo").Append("\n");
            }
            int[] triangles2 = mesh.GetTriangles(k);
            for (int l = 0; l < triangles2.Length; l += 3)
            {
                stringBuilder.Append(string.Format("f {0}/{0} {1}/{1} {2}/{2}\n", triangles2[l] + 1, triangles2[l + 1] + 1, triangles2[l + 2] + 1));
            }
        }
        return stringBuilder.ToString();
    }
}

```