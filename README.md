> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

# **视差遮挡贴图（Parallax Occlusion Mapping, POM）介绍**

视差遮挡贴图是视差贴图技术的高阶实现，通过‌**光线步进（Raymarching）算法**‌精确计算视线与高度图的交点，模拟复杂表面（如砖墙、岩石）的几何遮挡效果。相比标准视差贴图，POM能更真实地表现深度变化和自阴影，适用于高精度材质表现。

# **核心原理**

* ‌**分层深度检测**‌将视线方向在切线空间分解为多层（通常8-15层），逐层采样高度图，通过二分法逼\*视线与表面的交点。
* ‌**动态采样优化**‌根据视角与法线的夹角动态调整采样层数（\*\*行视角增加层数，垂直视角减少层数），\*衡精度与性能。
* ‌**遮挡关系重建**‌通过交点处的UV偏移修正纹理采样位置，模拟凹凸表面的光线遮挡效果。

---

# **Unity URP 实现示例与原理详解**

## **代码关键点解析**

* ‌**动态采样层数**‌

  + 通过`lerp(_MaxSamples, _MinSamples, saturate(dot(float3(0,0,1), viewDirTS)))`实现视角自适应分层，优化性能。
* ‌**光线步进循环**‌

  + 循环比较当前层高度与采样深度，找到首个交点区间，避免全精度遍历。
* ‌**二分法优化**‌

  + 在初步交点区间内使用`lerp`插值计算精确UV，减少采样次数。
* ParallaxOcclusionMapping.shader

  ```
  |  |  |
  | --- | --- |
  |  | Shader "Universal Render Pipeline/POM" |
  |  | { |
  |  | Properties |
  |  | { |
  |  | _MainTex("Albedo", 2D) = "white" {} |
  |  | _NormalMap("Normal Map", 2D) = "bump" {} |
  |  | _HeightMap("Height Map", 2D) = "white" {} |
  |  | _ParallaxScale("Height Scale", Range(0, 0.1)) = 0.05 |
  |  | _MinSamples("Min Samples", Int) = 8 |
  |  | _MaxSamples("Max Samples", Int) = 15 |
  |  | } |
  |  |  |
  |  | SubShader |
  |  | { |
  |  | Tags { "RenderPipeline"="UniversalPipeline" } |
  |  |  |
  |  | HLSLINCLUDE |
  |  | #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl" |
  |  |  |
  |  | TEXTURE2D(_MainTex);    SAMPLER(sampler_MainTex); |
  |  | TEXTURE2D(_NormalMap);  SAMPLER(sampler_NormalMap); |
  |  | TEXTURE2D(_HeightMap);  SAMPLER(sampler_HeightMap); |
  |  | float _ParallaxScale; |
  |  | int _MinSamples, _MaxSamples; |
  |  |  |
  |  | float2 ParallaxOcclusionMapping(float3 viewDirTS, float2 uv) |
  |  | { |
  |  | // 动态计算采样层数 |
  |  | int numSamples = (int)lerp(_MaxSamples, _MinSamples, saturate(dot(float3(0,0,1), viewDirTS))); |
  |  | float layerHeight = 1.0 / numSamples; |
  |  | float2 deltaUV = _ParallaxScale * viewDirTS.xy / viewDirTS.z / numSamples; |
  |  |  |
  |  | // 光线步进初始化 |
  |  | float currentLayerHeight = 0; |
  |  | float2 currentUV = uv; |
  |  | float currentDepth = 1 - SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, currentUV).r; |
  |  |  |
  |  | // 分层检测 |
  |  | [loop] |
  |  | for (int i = 0; i < 15; ++i) { |
  |  | if (currentLayerHeight >= currentDepth) break; |
  |  | currentUV -= deltaUV; |
  |  | currentDepth = 1 - SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, currentUV).r; |
  |  | currentLayerHeight += layerHeight; |
  |  | } |
  |  |  |
  |  | // 二分法精确交点 |
  |  | float2 prevUV = currentUV + deltaUV; |
  |  | float prevDepth = currentDepth - layerHeight; |
  |  | float weight = (currentLayerHeight - currentDepth) / (prevDepth - currentDepth); |
  |  | return lerp(currentUV, prevUV, weight); |
  |  | } |
  |  | ENDHLSL |
  |  |  |
  |  | Pass |
  |  | { |
  |  | HLSLPROGRAM |
  |  | #pragma vertex vert |
  |  | #pragma fragment frag |
  |  |  |
  |  | struct Attributes |
  |  | { |
  |  | float4 positionOS : POSITION; |
  |  | float2 uv : TEXCOORD0; |
  |  | float3 normalOS : NORMAL; |
  |  | float4 tangentOS : TANGENT; |
  |  | }; |
  |  |  |
  |  | struct Varyings |
  |  | { |
  |  | float4 positionCS : SV_POSITION; |
  |  | float2 uv : TEXCOORD0; |
  |  | float3 viewDirTS : TEXCOORD1; |
  |  | }; |
  |  |  |
  |  | Varyings vert(Attributes IN) |
  |  | { |
  |  | Varyings OUT; |
  |  | VertexPositionInputs posInput = GetVertexPositionInputs(IN.positionOS.xyz); |
  |  | OUT.positionCS = posInput.positionCS; |
  |  |  |
  |  | // 转换视角方向到切线空间 |
  |  | VertexNormalInputs normInput = GetVertexNormalInputs(IN.normalOS, IN.tangentOS); |
  |  | float3 viewDirWS = GetWorldSpaceViewDir(posInput.positionWS); |
  |  | OUT.viewDirTS = TransformWorldToTangent(viewDirWS, |
  |  | normInput.tangentWS, normInput.bitangentWS, normInput.normalWS); |
  |  |  |
  |  | OUT.uv = IN.uv; |
  |  | return OUT; |
  |  | } |
  |  |  |
  |  | half4 frag(Varyings IN) : SV_Target |
  |  | { |
  |  | // 计算POM偏移后的UV |
  |  | float2 pomUV = ParallaxOcclusionMapping(normalize(IN.viewDirTS), IN.uv); |
  |  |  |
  |  | // 采样最终纹理 |
  |  | half4 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, pomUV); |
  |  | half3 normalTS = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, pomUV)); |
  |  |  |
  |  | return half4(albedo.rgb, 1); |
  |  | } |
  |  | ENDHLSL |
  |  | } |
  |  | } |
  |  | } |
  ```

---

# **技术对比与性能建议**

| 特性 | 标准视差贴图 | 陡峭视差贴图 | POM |
| --- | --- | --- | --- |
| ‌**采样次数**‌ | 单次 | 5-15次 | 8-15次+二分法 |
| ‌**陡峭表面精度**‌ | 低 | 中 | 高 |
| ‌**适用\*台**‌ | 移动端 | PC/主机 | 高端PC |
| ‌**推荐参数**‌ | `Scale=0.02` | `Scale=0.05` | `Scale=0.03-0.07` |

## 实际应用中需注意：

* 高度图建议使用法线贴图的Alpha通道节省资源
* 过高的`_ParallaxScale`可能导致边缘拉伸，建议不超过0.1
* 移动端可减少`_MaxSamples`至8层以降低开销

---

> [【从UnityURP开始探索游戏渲染】](https://github.com):[豆荚加速器](https://doujiaa.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）
