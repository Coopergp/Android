---
APK体积优化
---
#### 目录
1. 资源优化
2. 代码优化
3. So库优化
##### 资源优化
1. 减少资源配置：Gradle 插件的 resConfigs 属性来移除应用不需要的语言资源
2. 图片压缩和转化：使用TinyPNG在线工具，有损压缩PNG和JPEG图像；使用 Android Studio 将现有 BMP、JPG、PNG 或静态 GIF 图片转换为 WebP 格式；
3. 无用资源移除：Lint检测冗余资源（动态加载的资源可能会被分析成冗余资源）
##### 代码优化
1. 代码混淆：在gradle文件里面打开配置：minifyEnabled true；在proguard-rules.pro 文件里面配置混淆规则
2. 冗余代码移除：Lint检测冗余代码(不能分析反射创建使用的类)
##### So库优化
适配市场上适配范围比较广的CPU架构，例如 armeabi-v7a 和 arm64-v8a
