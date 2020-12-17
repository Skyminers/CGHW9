# 作业八

## Light Up Your World

在之前的太阳系中添加了光照，包括：环境光、漫反射和镜面反射，其中镜面反射使用了 Cook-Torrance 。

### 环境光部分

通过给每个物体的颜色光照分量 `+0.1` 来模拟环境光

### 漫反射部分

计算入射光的方向向量和面的法向量，将两个向量点乘的结果作为漫反射的强度。

### 镜面反射部分

使用 Cook-Torrance, BRDF 模型。

## 实现方式

由于作为光源的太阳是不需要通过光照来计算其颜色的，所以太阳的着色器需要与其他物体的着色器分开实现。太阳的着色器与之前类似，较为简单。在其他物体的着色器中加入对光照计算的代码。

在片段着色器中会对点光线进行判断，需要使用到一些变量：
- 光源坐标：由主程序指定
- 光源颜色：由主程序指定
- 摄像机坐标：由主程序获取并指定
- Cook-Torranc 参数：由主程序指定
- 点坐标：在顶点着色器中取出点的世界坐标，将该坐标传入片段着色器中
- 法向量：由于所有的点都在球上，那么某一个点的法向量即该点与该点所在球心两点组成的向量。所以法向量可以由顶点着色器计算并传入片段着色器中。

Vertex Shader:

```C++
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aColor;

out vec3 ourColor;
out vec3 normal;
out vec3 worldPos;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main() {
    gl_Position = model * vec4(aPos, 1);
    worldPos = vec3(gl_Position);

    vec4 centerPosition = model * vec4(vec3(.0,.0,.0), 1);

    normal = vec3(gl_Position - centerPosition);
    gl_Position = projection * view * gl_Position;
    ourColor = aColor;
}
```

Fragment Shader:

```C++
#version 330 core
out vec4 FragColor;
in vec3 ourColor;
in vec3 normal;
in vec3 worldPos;

uniform vec3 lightColor;
uniform vec3 lightPos;
uniform vec3 cameraPos;

uniform float roughness, fresnel;

float cookTorranceSpec(vec3 lightDir, vec3 viewDir, vec3 surfaceNormal, float r, float f);
float beckmannDistribution(float NdotH, float f);

void main() {
    float globalAlpha = 0.1;
    vec3 global = lightColor * globalAlpha;

    vec3 norm = normalize(normal);
    vec3 lightDir = normalize(lightPos - worldPos);

    float diffuse = max(dot(normal, lightDir), 0.0);

    vec3 viewDir = normalize(cameraPos - worldPos);
    float speculat = cookTorranceSpec(
        lightDir,
        viewDir,
        norm,
        roughness,
        fresnel);

    vec3 result = (global + diffuse + speculat) * ourColor;

    FragColor = vec4(result, 1.0);//vec4(1.0, 0.5, 0.2, 1.0);
}

float cookTorranceSpec(vec3 lightDir, vec3 viewDir, vec3 surfaceNormal, float r, float f){
    float VdotN = max(dot(viewDir, surfaceNormal), .0);
    float LdotN = max(dot(lightDir, surfaceNormal), .0);

    vec3 H = normalize(lightDir + viewDir);

    float NdotH = max(dot(surfaceNormal, H), .0);
    float VdotH = max(dot(viewDir, H), 0.000001);
    float x = 2.0*NdotH / VdotH;
    float G = min(1.0, min(x*VdotN, x*LdotN));

    float D = beckmannDistribution(NdotH, r);

    float F = pow(1.0 - VdotN, f);

    return G*F*D/max(3.14159265*VdotN*LdotN, 0.000001);
}

float beckmannDistribution(float NdotH, float f){
    float NdotH2 = NdotH * NdotH;
    return exp( (NdotH2 - 1)/(f*f*NdotH2) ) / ( 3.14159265*f*f*NdotH2*NdotH2 );
}
```

## 最终效果

/截屏2020-12-17 下午3.44.28.png

/截屏2020-12-17 下午3.45.17 2.png

## 代码文件

- main.cpp : 主函数部分，包括初始化以及渲染部分，以及轮子的平移。对 `.obj` 文件的读取与显示(Polygonal Meshed)也写到了该文件中。
- Sphere.h/Sphere.cpp : 球形绘制类，该部分绘制一个球，并且将节点信息以及索引信息储存在`vector`当中并且返回首地址。
- Camera.h/Camera.cpp : 摄像机类，为了使main.cpp文件尽可能简洁，其中关于摄像机方向设置以及移动都在摄像机类中实现。
- Shaders.h/Shaders.cpp : 着色器类，该类封装了对于着色器的读取、编译、链接以及对于着色器变量的捕获修改。
- vertexShader.glsl : 参与光照计算的顶点着色器
- fragmentShader.glsl : 参与光照计算的片段着色器
- sunVertexShader.glsl : 太阳的顶点着色器
- sunFragmentShader.glsl : 太阳的片段着色器

## 其他说明

- 使用 Clion 完成项目
- 使用库： `glfw 3.3`, `glm 0.9.8.5`
- 操作系统为 macOS

`MAC`下请使用命令行运行该程序，在本地测试中，直接双击可执行文件会导致错误，需要使用命令行来执行。

程序执行时会使用四个.glsl外部文件，请将这四个着色器代码文件与程序放在同目录下执行。