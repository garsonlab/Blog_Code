---
title: 一个简单的着色器
date: 2016-9-23
tags:
- Shader
- Unity
categories: UnityShader
---

一直在关注[CandyCat的博客](http://blog.csdn.net/candycat1992)，所以在第一时间买了她新出的《Unity Shader 入门精要》，看了好久才记得要做读书笔记...是不是可以换句话说书写的太精彩忘记了
目前为止，第6章快看完了，回顾第五章讲的基础，就是一个简单的定点片元着色器，代码及注释如下：
``` cpp
Shader "Mine/5_SimpleShader"
{
	Properties{
		_Color ("Color Tint", Color) = (1.0,1.0,1.0,1.0)
	}
	SubShader{
		Pass{
			CGPROGRAM
				#pragma vertex vert
				#pragma fragment frag

				fixed4 _Color;
				//input
				struct a2v {
					float4 vertex : POSITION; // 用模型空间定点填充vertex
					float4 normal : NORMAL; // 用模型空间的法线填充
					float4 texcoord : TEXCOORD0; // 用第一套纹理填充
				};

				//vertex output
				struct v2f {
					float4 pos : SV_POSITION; // 裁剪空间中的位置信息填充pos
					fixed3 color : COLOR0; // 储存颜色信息
				};

				v2f vert(a2v v) { //SV_POSITION意思是输出的是裁剪空间坐标
					v2f o;
					o.pos = mul(UNITY_MATRIX_MVP, v.vertex); //转换到裁剪空间,顶点着色器中必须包含一个 SV_POSITION,否则无法得到裁剪空间中的坐标
					o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);//normal 范围（－1，1），color范围则成了（0，1）
					return o;
				}

				fixed4 frag(v2f i) : SV_Target { //SV_Target 把输出的颜色储存到默认帧缓冲
					return fixed4(i.color * _Color, 1.0);
				}

			ENDCG
		}
	}
}
```

还有，以后记得自己实践并记录！