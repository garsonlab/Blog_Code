---
title: 基础光照——漫反射与高光反射
date: 2016-9-23
tags:
- Shader
- Unity
categories: UnityShader
---

### 矩阵
继续学习《Unity Shader 入门精要》。渲染的流程前部分是坐标变换，变换顺序是： 
模型空间（Model Space）-->世界空间（World Space）-->观察空间（View Space）-->裁剪空间-->屏幕空间
具体的矩阵变换可以方便的使用内置矩阵：

	UNITY_MATRIX_MVP        当前模型视图投影矩阵
	UNITY_MATRIX_MV           当前模型视图矩阵
	UNITY_MATRIX_V              当前视图矩阵。
	UNITY_MATRIX_P              目前的投影矩阵
	UNITY_MATRIX_VP            当前视图*投影矩阵
	UNITY_MATRIX_T_MV       移调模型视图矩阵
	UNITY_MATRIX_IT_MV      模型视图矩阵的逆转
	UNITY_MATRIX_TEXTURE0   UNITY_MATRIX_TEXTURE3          纹理变换矩阵

记住变换顺序很有必要，并且，如果两个矩阵要做 mul 运算的时候，必须在同一坐标空间下。

### 漫反射光照模型
``` cpp

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

Shader "Mine/6_DiffuseVertexLevel" {
	Properties {
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
	}
	SubShader {
		Pass {
			Tags { "LightMode"="ForwardBase" } //只有定义了正确的 LightMode 才能得到要用的 _LightColor0 ...
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc" //得到要使用的 _LightColor0 等内置变量

			fixed4 _Diffuse;

			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f {
				float4 pos : SV_POSITION;
				fixed3 color : COLOR; // or TEXCOORD0
			};

			v2f vert(a2v v) {
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex); // must ,get projection space

				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz; //get ambient
				// 把法线从模型空间转换到世界空间，用的右乘当逆矩阵
				fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
				fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz );//只适用于平行光
				//漫反射 ＝ 入射光 ＊ 漫反射系数 ＊ max（0，dot（法线，光源方向））
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
				//如果是半lambert，则公式为
				//漫反射 ＝ 入射光 ＊ 漫反射系数 ＊ （缩放倍数＊dot（法线，光源方向） ＋ 偏移）
				//一般缩放倍数和便宜都取0.5
				//fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * (0.5*dot(worldNormal,worldLight)+0.5);

				o.color = ambient + diffuse;
				return o;
			}

			fixed4 frag(v2f i) : SV_Target {
				return (i.color, 1.0);
			}

			ENDCG			
		}
	}
	Fallback "Diffuse"
}

```

如果要改为逐像素光照，需要改变 struct v2f，
``` cpp
struct v2f {
	float4 pos : SV_POSITION;
	fixed3 worldNormal : TEXCOORD0
};
```
剩下的工作就是把 vert函数中的部分挪到 frag函数中。
另，代码中的半Lambert光照模型是为了解决在实际中，普通漫反射在光照不到的情况下是暗黑色的问题。

### 高光反射光照模型

``` cpp
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

Shader "Custom/6_SpecularVertexLevel" {
	Properties {
		_Diffuse ("Diffuse", Color) = (1.0, 1.0, 1.0, 1.0)
		_Specular ("Specular", Color) = (1.0, 1.0, 1.0, 1.0)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass {
			Tags { "LightMode"="ForwardBase" }

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"

			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;

			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f {
				float4 pos : SV_POSITION;
				fixed color : COLOR;
			};

			v2f vert(a2v v) {
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

				fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject)); //用unity内置函数是： ＝ UnityObjectToWorldNormal(v.normal)
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				//漫反射 ＝ 入射光 ＊ 漫反射系数 ＊ max（0，dot（入射方向，法线））
				fixed3 diffuse = _LightColor0.rgb * _Diffuse * max(0, dot(worldNormal, worldLightDir));
				
				//reflect 计算反射方向,两个参数是 入射方向、法线，但是要求入射方向是光源指向入射点，故取反
				fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
				//计算视觉方向首先是要求处于同一空间下，然后向量计算
				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
				//高光反射 ＝ 入射光 ＊ 高光系数 ＊ pow（max（0，反射方向 ＊ 视觉方向），光泽度）
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
				
				//如果是 BlinnPhong 则把反射方向换成 光线方向和视觉方向和
				//fixed3 halfDir = normalize(worldLightDir + viewDir);
				//fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(worldNormal,halfDir)), _Gloss);

				o.color = ambient + diffuse + specular;
				return o;		
			}

			fixed4 frag(v2f i) : SV_Target {
				return (i.color, 1.0);
			}

			ENDCG
		}
	}
	Fallback "Specular"
}
```

如果改为逐像素光照需改动 struct v2f
``` cpp
struct v2f {
	float4 pos : SV_POSITION;
	fixed worldNormal: TEXCOORD0;
	fixed worldPos: TEXCOORD1;
};
```

然后换上相应的即可。
代码中给出了用的最多的BlinnPhong模型，目前用的最多。

逐顶点光照和逐像素的区别在于计算的相对少和多，得到效果肯定计算多的逐像素更加显得平滑。