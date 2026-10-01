 

AIGC：标签："1"内容生产商：001191440300708461136T1XGW3生产商ID:0df9b94cb1f2b77a7591b85bebbd21b1_393defa8bd2c11f19ba 1525400638852ReservedCode1:bzBeH66zrI1HXRJzO9H7uI/7kke425mzDaGOTeAUo+N2nBBJj36oooaxRxAVGBwQnhSwe0NuIaTdE2hFEbkvBwj43CB3BCiWuK1vcamFpzFRNqmIFcuDemaSc4Wme74ZK+cgTuElVsKVH0Xgk8Vv+dSWhVt4H7WudSgFfYTvm8HyY2c/AjM53bF8us0= ContentPropagator:00119 1440300708461136T1XGW3PropagateID:0df9b94cb1f2b77a7591b85bebbd21b1_393defa8bd2c11f19ba 1525400638852ReservedCode2:bzBeH66zrI1HXRJzO9H7uI/7kke425mzDaGOTeAUo+N2nBBJ36ooaxRxAVGBwQnhSwe0NuIaTdE2hFEbkvBwj43CB3BCiWuK1vcamFpzFRNqmIFcuDemaSc4Wme74ZK+cgTuElVsKVH0Xgk8Vv+d SWhV T4H7Wu dSgFfYTvm8HyY2c/AjM53bF8us0=

「看世界」S1 交付说明 —— 实时取景视觉管线

S0已交付工程骨架；本阶段在S0基础上实现：Camerax实时取景+MediaPipe语义分割+GLES剪影合成取景器.云端无安卓工具链，本阶段交付为完整源码+本地构建与自测指引，不产出APK。

1. 本阶段目标

相机主页由占位替换为真实取景器，达到：

打开即见实时画面，人物皮肤区域实时变黑、外轮廓叠加白色亮边（默认纯白）；
画幅比例 5 档（4:3 / 16:9 / 9:16 / 1:1 / 3:4）切换仅重裁取景区域，相机不重绑、模型不重载、无卡顿；
前后摄切换（带黑屏遮罩过渡，前置自动镜像）；
闪光灯三态（关 / 自动 / 常亮），AUTO 随环境亮度自动开灯；
弱光检测（Y 平面亮度均值），光线不足时顶部徽标切换为"光线不足"；
模型加载状态徽标：加载中 → LIVE / 模型加载失败。

2.新增文件

 
app/src/main/java/com/堪仕捷/camera/
├--视力/
│├--YuvFrameScaler.KT#ImageProxy(YUV420_888)→256×256 RGB(跳读采样，避免全帧转换)
│├--SegmentationEngine.KT#MediaPipe ImageSegmenter(selfie_multiclass)，GPU优先+CPU回退
│└--MaskPostProcessor.kt#六类→皮肤二值、脸部膨胀填孔(严格模式近似)、EMA时域平滑
├--渲染/
│├--ShaderLibrary.kt#合成GLSL：皮肤变黑+亮边（纯白/自动取色/双线）
│└--GlPreviewRenderer.KT#GLSurfaceView.Renderer+Camerax SurfaceProvider(OES消费相机帧）
├--相机/
│├--CameraManager.kt#Cameraax绑定/前后摄重绑/闪光/弱光/推理管线调度
│└--FlashMode.kt#闪光三态枚举（相机层）
├-util/
│└--FpsMeter.kt#帧频统计（s7性能角恢复用）
└-ui/
├--相机/
│├--CameraScreen.kt#重写：GL取景器+顶部/底部控件+遮罩
│└--CameraViewModel.kt#UI状态（模型态/低光/镜头/比例/闪光）
└--组件/
├--AspectRatioPicker.kt#底部横向5档比例选择条
└--StatusBadge.kt#加载中/LIVE/光线不足/模型失败徽标
 

3. 实现要点

3.1 取景链路（核心）

 
相机预览(SurfaceProvider=GLRenderer)
│表面直接交给GLSurfaceView的SurfaceTexture
▼
gl线程：oes纹理←updateteximage→合成明暗器→屏幕
图像分析(YUV)--每帧--▶YuvFrameScaler(256)--▶图像分割器(VIDEO)
└--▶MaskPostProcessor(皮肤二值+平滑)--▶256×256亮度纹理
 

预览与图像分析均不设比例约束→两者同为传感器原生比例，mask与预览帧严格1:1对齐，亮边不错位；
比例裁剪全部在GL顶层完成(COVER+letterbox，多余区域纯黑），因此切比例零开销；
推理在单线程执行器、掩码结果经@Volatile 字段交给GL线程上传，允许半帧延迟；
GPU委派初始化失败自动回退CPU，徽标不变色（仅内部日志标注）。

3.2亮边参数

S1使用默认值（开、纯白、宽度4、强度25%），参数已暴露为渲染器@Volatile 字段，S5设置页直接接线即可。

3.3 严格脸部模式（近似）

默认"严格模式"下，face-skin 区域先做 2 轮形态学膨胀再并入皮肤掩码，把眉毛/眼睛/嘴唇等孔洞整体填黑；精确五官轮廓需 face_landmarker（后续阶段）。

4. 本地构建与自测

4.1 构建

 
CD勘世杰
./gradlew:app：组装调试编号首次需下载依赖与Gradle
./gradlew:app：安装调试#或Android Studio直接运行
 

前置：Android SDK37/jdk17；自测机型Android8.0(API26)及以上.

4.2 自测清单

 
 
 
 
 
 
 
 
 

4.3已知限制(S1)

推理分辨率固定256:SIZE_512档需级联模型/重训练(后续阶段)，S1不生效；
严格脸部为近似填充：眉毛/眼睛轮廓的精确整块填黑待 face_landmarker 接入（后续阶段）；
仅竖屏(ROTATION_0)：横屏方向适配后续阶段；
亮边参数为默认值：设置页 S5 接线后可调；
弱光阈值经验值40：可在摄影师调整；
GPU/CPU 推理耗时因机型差异较大，性能统计角标 S7 交付；
快门（拍照/录像）为占位，S2/S3接入。

5.下一阶段预告(S2)

图像捕捉拍照+合成后出图(GL读回/重渲染路径)、mediastore落盘、EXIF写入；
拍后预览页（九大页面之一）、照片格式/分辨率档位接入设置。 （内容由AI生成，仅供参考）
