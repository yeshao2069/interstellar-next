# 星际穿越

星际穿越是一款高画质次世代模拟飞行游戏，游戏内你掌心的玩具蜕变为星际战舰，透过电视开始一场惊险刺激的飞行旅行

<img src=./docs/images/home0.jpg width=600>

- 视频演示 [开源！全平台次世代飞行跑酷游戏《星际穿越》](https://www.bilibili.com/video/BV1NRwdehE7t)
- 在线体验 https://gamemcu.com/fly/
- 开源地址 https://github.com/gamemcu/interstellar-next/

> 本项目基于Cocos Creator 3.8.3，新版本引擎自定义管线可能不适配，谨慎升级！！

> 为了方便大家交流学习，我拉了一个微信交流群，可以先加这个微信gamemcu1，请注明来源，否则无法通过

## 技术亮点
1. 自定义高清渲染管线
2. 重构PBR材质系统
3. 高品质后期处理
4. 多边形GPU粒子系统
5. 高性能碰撞物理模拟
6. IK模拟
7. 音效系统设计
8. 指数级高度雾

## 代码结构
```bash
cinestation //镜头控制器，用于调试
components //自定义组件
controllers //游戏控制器，镜头、手、飞船等控制器
datas //游戏数据
managers //各类管理器
pipline-next //自定义高清管线 + 后期处理
statics //静态资源，贴图、模型
resources //动态资源，音频
```

## 渲染优化
### 1.1 场景渲染
<img src=./docs/images/before.jpg width=600>

直接把模型和材质拖入引擎你将得到上面上面这张图，其实效果还不错，但离真实感还有不少差距，主要问题是间接光、曝光和反射。

经过优化得到下面效果

<img src=./docs/images/after.jpg width=600>

主要做了以下几个工作
- 自定义高清渲染管线（开启float输出，支持颜色值大于1）
- 添加fxaa（优化版，边缘更锐利）、bloom（基于mipmapblur）、tonemap（拟合函数，性能更好）
- 添加自定义PBR材质（移除了复杂的光照计算，保留IBL部分并加入单次散射拟合，避免金属过黑）
- 支持自定义烘培光照贴图
- 扩展光照探针，支持自定义环境贴图（模拟环境反射）

### 1.2 人手渲染
<img src=./docs/images/hand-before.jpg width=600>

经过以上处理，手的光照已经比较真实（一开始尝试SSS，后来发现直接烘培在本项目中视觉上差异不大）

<img src=./docs/images/hand-after.jpg width=600>

添加间接光（烘培贴图），可以让手看起来更真实，手的捏合也可以通过动态控制间接光的强度来模拟光影变化

至此，恭喜你！已完成主场景的渲染优化！

## 互动优化

### 2.1 静置的时候，让手动起来可以让画面更真实，可是该如何动呢？

<video src="./docs/videos/hand.mp4" autoplay muted loop controls width=600></video>

项目中通过叠加多个perlin噪声生成更平滑的分型噪声（FBM），并给手的肩关节、肘关节、腕关节不同的噪声幅度，模拟人的的呼吸感和摆动（这里考虑过K动画的形式实现，复杂度远高于加噪声）

<video src="./docs/videos/IK.mp4" autoplay muted loop controls width=600></video>

游戏中玩家可以操控手的摆动，这里是通过添加骨骼权重并动态控制关节旋转实现的，除了摆动其实还有个抓取动画，这里有个cocos使用的小技巧，默认骨骼动画是不能在引擎内编辑的，但是如何你将SkeletalAnimation替换成Animation，我们就可以利用cocos的动画编辑器编辑骨骼动画了，还可以动态调整抓取速度，非常方便

<video src="./docs/videos/tv.mp4" autoplay muted loop controls width=600></video>

电视里的画面是通过UI相机渲染出来的，发现只要给激光简单做下位移和旋转，画面看起来会有3D的感觉，有点小惊喜，这样我不需要重复渲染3D画面，可以省下很大的性能开销

为了让老式电视更加真实，项目中加入了扫描线的特效和暗角，后期效果中也加入了扫描线效果，可以在片段着色器中简单通过sin函数实现，然而这样非常耗性能，我写了个拟合函数
```c 
0.9+1.03*mix(-y*y,y*(y-2.)+1.,y)替代 
0.9+0.1*sin(y*500.0)
``` 
性能提升非常大（超级干货，可以抄！）

### 2.2 真实感尾焰
写实风格的尾焰过程挺坎坷，尝试了多种方案，贴图流动以及播放视频，不是效果不行就是性能不行

<video src="./docs/videos/tail-lensfare.mov" autoplay muted loop controls width=600></video>

最后通过
**多层模型+顶点偏移+噪声+颜色混合**
实现的，另外我们还加入了镜头光晕的效果，随着镜头的旋转，光晕会动态变化，具体可参考flame.effect的实现

<video src="./docs/videos/tail.mov" autoplay muted loop controls width=600></video>

### 2.3 多边形GPU粒子（不知道怎么取名字合适，先叫这个吧）
游戏内粒子的效果比较抓人眼球，相信很多人都想知道如何实现的，下面介绍下原理，具体实现请参考initParticleMesh函数的实现

<video src="./docs/videos/ps1.mov" autoplay muted loop controls width=600></video>

飞机收集电池的特效分成多段，由 **扫光+爆炸粒子+收集粒子** 组成

扫光是通过fresnel分段偏移实现的，爆炸粒子使用了常规的粒子系统，收集粒子比较复杂，其实是个模型，具体原理和飞机的合成特效一样，只是shader中做了不同处理

<video src="./docs/videos/ps2.mov" autoplay muted loop controls width=600></video>

这样的粒子实现具体分为四步
- 模型拆面，移除indices，计算每个面的重心坐标，并写入贝塞尔曲线参数
- 顶点着色器中，将顶点位置以重心坐标为参考，按照贝塞尔曲线进行偏移
- 加入噪声让顶点位置在贝塞尔曲线上随机偏移，可产生拉伸效果
- 在片段着色器中，根据粒子生命周期，将颜色、AO、反射做适当的处理

>有没人对电池效果的实现好奇？这里电池由两部分组成，分别是外壳、外发光和里面的闪电，这里外壳和外发光其实只是一张朝向相机的图片，里面的闪电是个模型，哈哈哈，极致优化～

### 2.3 指数级高度雾（不仅可以美颜，还可以抗锯齿）
<img src=./docs/images/nofog.jpg width=600>

这是没有雾气的效果，为了让大家直观感受指数级高度雾的效果（我只能把这种丑陋的图片拿出来）

<img src=./docs/images/fog.jpg width=600>

这是指数雾的效果，改善了很多，画面变干净了，锯齿也少了，但是场景缺乏层次

<img src=./docs/images/game1.jpg width=600>

<img src=./docs/images/game2.jpg width=600>

这是指数级高度雾的效果，画面不仅干净了，还更有层次了，也更有氛围感了

### 2.4 多种物理碰撞模拟
<img src=./docs/images/collide1.gif>
<img src=./docs/images/collide2.gif>
<img src=./docs/images/collide3.gif>

为了让游戏中的碰撞更加真实，针对不同的碰撞物，游戏内都做了不同处理，尾焰也会做不同程度的缩放

### 2.2 游戏跑了一段距离之后，模型出现破面！！？？

<img src=./docs/images/broken.jpg width=600>

由于第一次做跑酷游戏，所以在游戏跑了一段距离后，模型出现破面我惊呆了！！？？还有这种事，一度以为是引擎问题，难不成cocos给我做了自动lod？

后来发现是GPU浮点数精度问题，需要修改两个地方
- 将顶点着色器中的"precision mediump float;"改为"precision highp float;"否则距离超过100m就会破面
- 飞船移动超过100m，复位到起始点，并且场景内所有物体相对飞船一起平移，这里要处理好特效的位置

## 总结
由于篇幅关系，游戏内还有不少细节，本文没有涉及，大家可以去源码中挖掘

本项目从管线到材质到各种trick，通过对大量细节的打磨，最终呈现出了一个让人相对满意的画面和体验，希望本项目的代码和经验可以帮助到更多开发者