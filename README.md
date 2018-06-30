---
layout:     post
title:      用Unity实现直接光照部分的PBR
subtitle:   
date:       2018-06-28
author:     Richbabe
header-img: img/u3d技术博客背景.jpg
catalog: true
tags:
    - 计算机图形学
    - Unity
---
# 前言
之前我的两篇博客：[PBR原理](http://richbabe.top/2018/06/18/PBR%E5%8E%9F%E7%90%86/)和[探究PBR的两种流程以及Unity中的PBS](http://richbabe.top/2018/06/25/%E6%8E%A2%E7%A9%B6PBR%E7%9A%84%E4%B8%A4%E7%A7%8D%E6%B5%81%E7%A8%8B%E4%BB%A5%E5%8F%8AUnity%E4%B8%AD%E7%9A%84PBS/)已经简述了PBR的基本概念，现在让我们来看看如何用Unity实现直接光照部分的PBR。

# 反射率方程
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%8F%8D%E5%B0%84%E7%8E%87%E6%96%B9%E7%A8%8B.png?raw=true)
这是PBR的核心，翻译成自然语言，大概是：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%8F%8D%E5%B0%84%E7%8E%87%E6%96%B9%E7%A8%8B%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80.png?raw=true)
先解释下这个公式遗留的部分。半球积分∫，表示的是多光源下光照的叠加。之所以非要写成半球积分而不是 ∑，是为了兼容环境光照。如果你只考虑单个不衰减的直线光照的话，这部分其实可以直接去掉（并不是说数学上可以直接化简，而是因为这是一个特例）：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%8E%BB%E7%A7%AF%E5%88%86.png?raw=true)
看到这个lightDir • normal大家都应该很熟悉，如果将镜面反射系数设定为0，漫反射系数设定为1，公式就和单纯的Lambert漫反射基本一致：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/Lambert.png?raw=true)
不一致的部分是这个除π。因为它把亮度除低了，就只能相应调高光源的亮度补回来。看似别扭，但是回头一想，光源的亮度，难道不就应该比周围的物品高上很多吗？因为即使是直射，也还是会有很多光线被散射到其他方向，只有少部分才正常投射到了人眼中，漫反射的性质就是如此，之前不除π的做法其实才是错误的。

至于为什么除的是π，是因为：

![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/gs1.png?raw=true)

如果散射的光线最后都能汇集到一点的话，积分的结果就是会再乘一个π。所以分散的时候就需要除π。）

另外还有一个地方容易让人迷惑，按说经过半球积分汇集了不同方向的光线后，返回的结果应该是辐照度E（每单位面积），而这个反射率公式左边却是L（每单位角单位面积），这在单位上就说不过去。

实际上，是因为这个公式经过了化简，把一些中间参数给约掉了，剩下的部分形成了这样的结构。这篇文章有推导过程：[PBR Step by Step（三）BRDFs](http://www.cnblogs.com/jerrycg/p/4932031.html)

从“非数学”的角度考虑的话，也可以认为是这个单位面积汇集的不同方向的光线最后都融合并反射了出去，我们从中重新取了一条光线作为结果。

# BRDF
## 微平面
微表面模型是对现实物理光照的一种模型描述。

除了之前提到的是否金属会影响高光外，表面粗糙程度也会影响高光。
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%BE%AE%E5%B9%B3%E9%9D%A2%E7%A4%BA%E6%84%8F.png?raw=true)

## Cook-Torrance
根据该理论，可以认为物体的表面由无数不规则的镜面组成。那么高光将由各表面的法向分布有关，其法向量和灯光方向（I）与视角方向（V）的半角向量（H）接近的越多则高光越强（如图a）。我们用D（Normal Distribution Function/法线分布函数）表示。

同时考虑到光线被遮挡（如图b、c）的情况，用G（Geometry Function/几何遮蔽函数）表示。

而高光的反射比例由角度的变化而不同，用F（Fresnel Rquation/菲涅尔方程）表示。

综合以上三种因素高光反射可整理为：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/BRDF.png?raw=true)

这部分是个叫做Cook-Torrance的BRDF光照公式，具体推导过程可以看看这篇文章：[基于物理着色：BRDF](https://zhuanlan.zhihu.com/p/21376124)。分母的原理我这里就不叙述了（因为不会），我们主要来看看分子部分的实现。

## 镜面高光：法线分布函数 Normal Distribution Function
输入参数为：normal，h，粗糙度

这里和传统的BlinnPhong高光模型一样，是用半角向量h，也就是viewdir和lightdir的中间向量h，和normal求点乘来决定高光亮度的。

这里我的法线分布函数使用Trowbridge-Reitz GGX计算：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/D.png?raw=true)

Shader中的实现为：

```
//计算法线分布函数D（法向量，半角向量，粗糙度）
			fixed DistributionGGX(fixed3 N, fixed3 H, fixed roughness)
			{
				fixed a = roughness * roughness;
				fixed a2 = a * a;
				fixed NdotH = max(dot(N, H), 0.0);
				fixed NdotH2 = NdotH*NdotH;

				fixed nom = a2;//分子
				fixed denom = (NdotH2 * (a2 - 1.0) + 1.0);
				denom = UNITY_PI * denom * denom;//分母

				return nom / denom;
			}
```

这个公式使得不会辐射出多余的光，D不会大于1/π（除π的原因和上面漫反射部分一致）。当α非常接近0的时候，光照集中在一点，其他方向会完全看不到光线。这是符合现实的。
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/Dsa.png?raw=true)

## 几何遮蔽：几何函数 Geometry function
输入参数为：normal，viewDir，lightDir，粗糙度

这是一个其他传统光照模型不具有的特征，体现了光在物体粗糙面上反射时的损耗。
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/G1.png?raw=true)

![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/G2.png?raw=true)

这里我使用Smith’s Schlick-GGX实现：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/G.png?raw=true)

其中，直接光照时：

![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7k.png?raw=true)

间接光照（IBL）时：

![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E9%97%B4%E6%8E%A5%E5%85%89%E7%85%A7.png?raw=true)

效果就是粗糙度越大，亮度越低。但视线和光线越接近垂直，受粗糙度的影响就越小，合情合理。

k的取值范围都在逐渐逼近1/2。而直接光和间接光的差别是，直接光至少有1/8的吸收系数保底，而间接光没有。这是为了让完全光滑的物体，也能至少吸收一些光线。完全不吸收光线的物体是不应该存在的。

在这里我使用了直接光照的k,Shader实现为：


```
			//计算几何遮蔽函数G1（向量与法向量夹角值，粗糙度）
			fixed GeometrySchlickGGX(fixed NdotV, fixed roughness)
			{
				fixed r = (roughness + 1.0);
				fixed k = (r * r) / 8.0;

				fixed nom = NdotV;
				fixed denom = NdotV * (1.0 - k) + k;

				return nom / denom;
			}

			//计算双向几何遮蔽函数G（法向量，视线方向，入射方向，粗糙度）
			fixed GeometrySmith(fixed3 N, fixed3 V, fixed3 L, fixed roughness)
			{
				fixed NdotV = max(dot(N, V), 0.0);
				fixed NdotL = max(dot(N, L), 0.0);
				fixed ggx2 = GeometrySchlickGGX(NdotV, roughness);
				fixed ggx1 = GeometrySchlickGGX(NdotL, roughness);

				return ggx1 * ggx2;
			}
```

## 菲涅尔方程：Fresnel equation
输入参数为：normal,viewDir，F0(法向量与视线方向夹角为90度时的反射率)

菲涅尔方程以前一般是用在水体上的，因为水体粗糙度低反光能力强，却又不是金属，是菲涅尔效应最明显的现实物体。

在这里我使用的是Fresnel-Schlick近似(Fresnel-Schlick Approximation)：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/F.png?raw=true)

注意：这个公式和光照方向无关。

法线和视线夹角越大（视线越接近水平），F的值也就越大，反射光的亮度也越高，这就是所有物体都具有的菲涅尔效应。即使不是金属物体，在这种情况下都会产生和金属物体类似的表现。而当物体本身就是金属的时候(F0接近1)，不管视线是什么情况，F的值都会接近于1，那么菲涅尔效应也就看不出来了。

这看似是个无关紧要的特性——那只是我们大多没有意识到“物体应该如此”而已，但即使我们没注意到，我们的大脑却会依然会得出一个“不真实”的结论。其实菲涅尔效应的模拟比我们想象中要更重要，并不仅仅是在水体模拟这个情景下。

然而，对于金属物体而言，菲涅尔其实并不完全适用。他的F0参数对不同颜色值的反射率是不同的，而且还需要和表面颜色相乘，否则我们的大脑就会通知我们它“不像金属”，所以最终的做法是做这样一次处理：

F0 = mix(vec3(0.04), 表面颜色, 金属度);

这样代入公式的结果就比较符合金属的物理特征，而非金属由于F0值偏低，即使乘了表面颜色影响也不大。

注意这里的表面颜色仅仅是给金属物体用的，用于表现金属物体的特殊性质，高光部分本身并不需要和物体的表面颜色相乘。

Shader实现如下：

```
//计算菲涅尔方程F（法向量与视线方向的夹角，法向量与视线方向夹角为90度时的反射率）
			fixed3 fresnelSchlick(fixed3 N, fixed3 V, fixed3 F0)
			{
				float cosTheta = max(dot(N, V), 0.0);
				return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
			}
```
在片段着色器中F0计算方法为：

```
fixed3 F0 = lerp(fixed3(0.04,0.04,0.04), albedo, _Metalness);//金属与非金属的区别
```
## 整合DGF并计算
最后看回这个公式：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%8F%8D%E5%B0%84%E7%8E%87%E6%96%B9%E7%A8%8B.png?raw=true)

最后还有两个参数没有解明，也就是Kd（漫反射比例）和Ks（镜面反射比例）。

Ks（镜面反射比例）实际上就是F。之前的公式其实并不妥当，因为Ks和F其实是重复的，只需要乘一次。所以应该是：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E5%8E%BB%E6%8E%89ks.png?raw=true)

而Kd（漫反射比例），则是(1-F)(1-金属度)，除了需要减掉F外，还要再乘一次(1-金属度)。这是因为金属会更多的吸收折射光线导致漫反射消失，这是金属物质的特殊物理性质。在片段着色器中，Kd计算为：

```
fixed3 kD = (1.0 - fresnel) * (1.0 - _Metalness);//漫反射比例
```
在片段着色器中将DGF整合起来求最终的光照颜色：

```
                fixed NDF = DistributionGGX(normal, halfDir, _Roughness);//Cook-Torrance 的d项

				fixed G = GeometrySmith(normal, viewDir, lightDir, _Roughness);//Cook-Torrance 的g项

				fixed3 F0 = lerp(fixed3(0.04, 0.04, 0.04), albedo, _Metalness);//金属与非金属的区别
				fixed3 fresnel = fresnelSchlick(normal, viewDir, F0);//菲涅尔项

				fixed3 specular = NDF * G * fresnel / (4.0 * max(dot(normal, viewDir), 0.0) * max(dot(normal, lightDir), 0.0) + 0.001);//镜面反射部分 ps：+0.001是为了防止除零错误

				specular += lerp(specular,reflection,fresnel);

				fixed3 kD = (1.0 - fresnel) * (1.0 - _Metalness);//漫反射比例

				float4 sh = float4(ShadeSH9(half4(normal,1)),1.0);

				fixed3 Final = (kD * albedo + specular) * _LightColor0.xyz * (max(dot(normal, lightDir), 0.0) + 0.0);//镜面反射及diffuse部分整合

				return  float4(Final,1.0) + 0.03 * sh * albedo;//补个环境反射的光
```

# 运行效果
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/%E6%95%88%E6%9E%9C.gif?raw=true)

可以看到通过修改Metallic的值（0 ~ 1实现了非金属到金属的切换）

# DGF其他公式
其实，刚才说的这几个DGF公式都不是唯一的，因为这些公式即使是基于物理的，也还是会包含一些“只要和结果差不多就可以”的部分（比如那个1/8），因为严格的公式往往会为了不明显的细节而消耗大量计算时间，不值得。

所以，他们其实也都只是“并非那么拟合”的拟合公式。

而这几个公式，也有一些精度更低，但性能更好的拟合版本，诸如UE4的Paper里，菲涅尔部分使用的是这样一个神奇的公式：
![image](https://github.com/Richbabe/Richbabe.github.io/blob/master/img/PBR/%E7%94%A8Unity%E5%AE%9E%E7%8E%B0%E7%9B%B4%E6%8E%A5%E5%85%89%E7%85%A7%E9%83%A8%E5%88%86%E7%9A%84PBR/ue4-F.png?raw=true)
这个公式是用曲线拟合方式对之前那个菲涅尔方程的近似，通过把pow函数换成exp2，得到了更好一点的性能。

（是的，exp2比pow快，因为
```math
x^y = e^{y \ln x}
```
）

下列博客中有DGF多个公式的总结：
* [法线分布函数D](http://www.resetoter.cn/?p=577)
* [几何遮蔽函数G](http://www.resetoter.cn/?p=592)
* [菲涅尔方程F](http://www.resetoter.cn/?p=615)

# 结语
到这里，我们已经用Unity实现了PBR的直接光照部分，但是PBR不仅仅只有直接光照，他还有IBL（Image-Based Lighting) 基于纹理的光照，这里Mark了一些有用的博客：
* [Diffuse irradiance](https://learnopengl.com/PBR/IBL/Diffuse-irradiance)
* [LearnOpenGL - Specular IBL](https://learnopengl.com/PBR/IBL/Specular-IBL)
* [猴子都能看懂的PBR](https://zhuanlan.zhihu.com/p/33464301)

等我把《Real-Time Rendering》看完再来填这个坑把~

本博客的代码和资源均可在我的[github](https://github.com/Richbabe/UnityPBR)上下载，别忘了点颗Star哟！





