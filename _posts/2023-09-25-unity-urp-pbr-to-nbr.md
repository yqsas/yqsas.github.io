---
layout: post
title: "Unity URP Shader PBR 转 BlinnPhong"
subtitle: ''
author: "yqsas"
header-style: text
catalog: false
tags:
  - Unity
  - URP
---

## Unity URP Shader PBR 转 BlinnPhong

项目中有一些第三方或者ASE生成的Shader代码，默认都是用的PBR光照，然而大部分材质仅需类似SimpleLit中使用的BlinnPhong光照模型就够用了，尤其在移动端我们可以考虑优化。  
如何使用最少的代码，将PBR使用的Metallic流程的材质参数快速转换到BlinnPhong光照呢，今天研究了一会儿效果还行，供大家参考。

### Shader

```
//BlinnPhong额外加强高光，贴进PBR效果。
half4 UniversalFragmentBlinnPhongAddSpecular(InputData inputData, half3 diffuse, half4 specularGloss, half smoothness, half smoothnessPower, half3 emission, half alpha)
{
    // To ensure backward compatibility we have to avoid using shadowMask input, as it is not present in older shaders
    #if defined(SHADOWS_SHADOWMASK) && defined(LIGHTMAP_ON)
    half4 shadowMask = inputData.shadowMask;
    #elif !defined (LIGHTMAP_ON)
    half4 shadowMask = unity_ProbesOcclusion;
    #else
    half4 shadowMask = half4(1, 1, 1, 1);
    #endif

    Light mainLight = GetMainLight(inputData.shadowCoord, inputData.positionWS, shadowMask);

    #if defined(_SCREEN_SPACE_OCCLUSION)
        AmbientOcclusionFactor aoFactor = GetScreenSpaceAmbientOcclusion(inputData.normalizedScreenSpaceUV);
        mainLight.color *= aoFactor.directAmbientOcclusion;
        inputData.bakedGI *= aoFactor.indirectAmbientOcclusion;
    #endif

    MixRealtimeAndBakedGI(mainLight, inputData.normalWS, inputData.bakedGI);

    half3 attenuatedLightColor = mainLight.color * (mainLight.distanceAttenuation * mainLight.shadowAttenuation);
    half3 diffuseColor = inputData.bakedGI + LightingLambert(attenuatedLightColor, mainLight.direction, inputData.normalWS);
    #if defined(_SPECGLOSSMAP) || defined(_SPECULAR_COLOR) || defined(_MASKMAP)
    half3 specularColor = LightingSpecular(attenuatedLightColor, mainLight.direction, inputData.normalWS, inputData.viewDirectionWS, specularGloss, smoothness) * (smoothnessPower * 16);
    #endif

    #ifdef _ADDITIONAL_LIGHTS
    uint pixelLightCount = GetAdditionalLightsCount();
    for (uint lightIndex = 0u; lightIndex < pixelLightCount; ++lightIndex)
    {
        Light light = GetAdditionalLight(lightIndex, inputData.positionWS, shadowMask);
    #if defined(_SCREEN_SPACE_OCCLUSION)
            light.color *= aoFactor.directAmbientOcclusion;
    #endif
        half3 attenuatedLightColor = light.color * (light.distanceAttenuation * light.shadowAttenuation);
        diffuseColor += LightingLambert(attenuatedLightColor, light.direction, inputData.normalWS);
    #if defined(_SPECGLOSSMAP) || defined(_SPECULAR_COLOR) || defined(_MASKMAP)
        specularColor += LightingSpecular(attenuatedLightColor, light.direction, inputData.normalWS, inputData.viewDirectionWS, specularGloss, smoothness) * (smoothnessPower * 16);
    #endif
    }
    #endif

    #ifdef _ADDITIONAL_LIGHTS_VERTEX
    diffuseColor += inputData.vertexLighting;
    #endif

    half3 finalColor = diffuseColor * diffuse + emission;

    #if defined(_SPECGLOSSMAP) || defined(_SPECULAR_COLOR) || defined(_MASKMAP)
    finalColor += specularColor;
    #endif

    return half4(finalColor, alpha);
}

//metallic、smoothness 均为采样后已混合好的结果值
half4 MetallicBlinnPhong(InputData inputData, half3 albedo, half3 specularColor, half metallic, half smoothness, half3 emission, half alpha)
{
    const half smoothnessPower = smoothness;
    smoothness = exp2(10 * smoothness + 1); //SimpleLit做法
    const half oneMinusReflectivity = OneMinusReflectivityMetallic(metallic);
    albedo = albedo * oneMinusReflectivity;
    half3 specular = lerp(kDieletricSpec.rgb, albedo, metallic) * specularColor;

    half4 color = UniversalFragmentBlinnPhongAddSpecular(
        inputData,
        albedo,
        half4(specular.rgb, 0),
        smoothness,
        smoothnessPower,
        emission,
        alpha);
    return color;
}
```

### 对比

对比图偷懒就不放了🫣。

我们以 URP 自带的地形着色器 `Universal Render Pipeline/Terrain/Lit` 举个例子。

- 修改前：
```
half4 color = UniversalFragmentPBR(inputData,
                                       albedo,
                                       metallic,
                                       /* specular */ half3(0.0h, 0.0h, 0.0h),
                                       smoothness,
                                       occlusion,
                                       /* emission */ half3(0, 0, 0),
                                       alpha);
```

- 修改后
```
half4 color = MetallicBlinnPhong(
                    inputData,
                    albedo,
                    1,
                    metallic,
                    smoothness,
                    half3(0, 0, 0),
                    alpha);
```



