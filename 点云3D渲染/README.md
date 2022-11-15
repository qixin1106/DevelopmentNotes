# PointCloud for iOS 点云数据渲染

### 前言

我们要实现的是一个机器人实时扫描到的点云数据展示视图,它提供一些接口,如获刷新云的数据/机器人坐标数据等.

- 我们选择使用SceneKit+MSL来实现展示点云,机器人及一些辅助模型.
- 需要设计同事给出对应的机器人低模,并导入到场景中.
- 我们还需要根据产品的需求设计一些自定义的交互操作.
以上问题通过不断踩坑.最终实现了效果.

### 技术概述

SceneKit苹果官方的概述
> SceneKit combines a high-performance rendering engine with a descriptive API for import, manipulation, and rendering of 3D assets. Unlike lower-level APIs such as Metal and OpenGL that require you to implement in precise detail the rendering algorithms that display a scene, SceneKit requires only descriptions of your scene’s contents and the actions or animations you want it to perform.

翻译
> SceneKit 将高性能渲染引擎与用于导入、操作和渲染 3D 资产的描述性 API 相结合。 与 Metal 和 OpenGL 等较低级别的 API 不同，这些 API 需要您精确地实现显示场景的渲染算法的细节，SceneKit 只需要场景内容的描述以及您希望它执行的动作或动画。

### 如何导入外部模型

SceneKit支持的外部模型是obj和dae格式,我们为了兼容andorid的库选择了obj.
你可以把obj模型拖入到你的scn文件中.

> scn就是一个可视化的3D场景文件.

这时你可以从左侧的列表中看到你的模型名字,我们改为一个自定义名字,这个名字后面会用到.

**下面来到Coding**

```objc
// 先把你的资源scn文件转化为SCNScene对象
NSURL *bundlePathUrl = [[NSBundle mainBundle] bundleURL];
bundlePathUrl = [bundlePathUrl URLByAppendingPathComponent:@"robot.scn"];
SCNScene *scene = [SCNScene sceneWithURL:bundlePathUrl options:nil error:nil];

// 从SCNScene中获取你刚导入的模型节点
SCNNode *robotModelNode = [scene.rootNode 

// "recursively"设置为false,因为我们不需要遍历子节点,这个模型在我设计时,就是挂在root下面的.
childNodeWithName:@"robotModel" recursively:NO];

// 因为只需要展示白膜,所以并未增加材质贴图,直接设置白色即可
robotModelNode.geometry.firstMaterial.diffuse.contents = [SCNColor whiteColor];

// 这里有个坑,因为设计软件里的坐标系和SceneKit的坐标系不一致,导致这里需要再修复一下.
robotModelNode.eulerAngles = SCNVector3Make(M_PI_2, 0, M_PI_2);

// 设置模型坐标,因为你在scn中的节点位置可能并不是原点.
robotModelNode.position = SCNVector3Make(0, 0, 0);
```
> 你可以把你的导入模型都放入一个模型资源的scn中,后面可以通过这个方法来获取自定义模型.

现在你应该可以在场景中看到你的模型了.

### 点云数据PCD-Point Cloud Data

#### *从ROS订阅接收点云*

首先从ROS这块的订阅的点云数据回调开始
有几个重要的变量需要获取
- point_step
> 一个点在data中所占的长度

- fields
> 对每个点的描述,其中包含对数据类型,偏移量等等.后面解析时会用到

- data
> 点云的真实数据

```objc
void pointCloudCallback(const sensor_msgs::PointCloud2ConstPtr msg)
{   
    // point_step
    uint32_t point_step = msg->point_step;
    // data
    NSData *data = [NSData dataWithBytes:msg->data.data() length:msg->data.size()];
    // fields
    NSMutableArray *fields = @[].mutableCopy;
    for (int i = 0; i < msg->fields.size(); i++) {
        sensor_msgs::PointField pointField = msg->fields[i];
        std::string name = pointField.name;
        NSString *ocName = [[NSString alloc] initWithCString:name.c_str() encoding:NSUTF8StringEncoding];
        
        NSMutableDictionary *dict = @{}.mutableCopy;
        [dict setValue:ocName forKey:@"name"];
        [dict setValue:@(pointField.offset) forKey:@"offset"];
        [dict setValue:@(pointField.datatype) forKey:@"datatype"];
        [dict setValue:@(pointField.count) forKey:@"count"];
        [fields addObject:[dict copy]];
    }
    
    // 用来接收点云数据的模型
    SRPointCloudInfo *info = [[SRPointCloudInfo alloc] init];
    info.pointData = data;
    info.pointStep = point_step;
    info.fields = [fields copy];
}
```
> 注意点云数据是一次性推送过来的,其中data是指所有点云的数据.

#### *解析点云并转成模型显示在场景中*

我们在SceneKit中使用创建一个SCNNode节点,用来接收显示点云

```objc
self.pcdNode = [SCNNode node];
[self.worldNode addChildNode:self.pcdNode];
```
但是这个节点是个空节点,我们目前还没有多边形绑定.我们需要一个SCNGeometry对象绑定到节点上,这样我们才能渲染出自己想要的形状.

```objc
SCNGeometry *pointGeometry = [self handleNewPointGeometryWithData:pointCloudInfo];
self.pcdNode.geometry = pointGeometry;
```
那么下面就看下如何创建SCNGeometry对象了.

```objc
/// 处理点云数据转成多边形
- (SCNGeometry *)handleNewPointGeometryWithData:(SRPointCloudInfo*)pointCloudInfo
{
    NSInteger xOffset = [pointCloudInfo.fields[0][@"offset"] integerValue];
    NSInteger yOffset = [pointCloudInfo.fields[1][@"offset"] integerValue];
    NSInteger zOffset = [pointCloudInfo.fields[2][@"offset"] integerValue];
    
    NSData *data = pointCloudInfo.pointData;
    NSInteger count = data.length / pointCloudInfo.pointStep;
    
    // 解析
    NSInteger index = 0;
    SCNVector3 *vectors = malloc(sizeof(SCNVector3)*count);
    SCNVector3 *colors = malloc(sizeof(SCNVector3)*count);
    for (int k = 0; k < data.length; k+=pointCloudInfo.pointStep)
    {
        @autoreleasepool {
            float x,y,z;
            [data getBytes:&x range:NSMakeRange(k+xOffset, 4)];
            [data getBytes:&y range:NSMakeRange(k+yOffset, 4)];
            [data getBytes:&z range:NSMakeRange(k+zOffset, 4)];
            
            SCNVector3 vector = SCNVector3Make(x, y, z);
            memcpy(vectors+index, &vector, sizeof(SCNVector3));
            
            // 这里负责记录颜色数据,但是目前我们并没有发送这块的数据.
            // 所以暂时写死,后面点云上色的功能后面再说吧.
            SCNVector3 color = SCNVector3Make(arc4random()%256/255.0, arc4random()%256/255.0, arc4random()%256/255.0);
            memcpy(colors+index, &color, sizeof(SCNVector3));
        }
        index++;
    }

    //顶点
    SCNGeometrySource *pointSource = [SCNGeometrySource geometrySourceWithVertices:vectors count:count];
    SCNGeometryElement *element = [SCNGeometryElement geometryElementWithData:nil primitiveType:SCNGeometryPrimitiveTypePoint primitiveCount:count bytesPerIndex:0];
    
    //颜色
    NSData *colorsData = [NSData dataWithBytes:colors length:sizeof(SCNVector3) * count];
    SCNGeometrySource *colorSource = [SCNGeometrySource geometrySourceWithData:colorsData semantic:SCNGeometrySourceSemanticColor vectorCount:count floatComponents:YES componentsPerVector:3 bytesPerComponent:sizeof(float) dataOffset:0 dataStride:sizeof(SCNVector3)];

    //创建几何体
    SCNGeometry *geometry = [SCNGeometry geometryWithSources:@[pointSource, colorSource] elements:@[element]];
    
    //清理
    free(vectors);
    vectors = NULL;
    free(colors);
    colors = NULL;

    //设置Shader
    SCNProgram *shaderProgram = [SCNProgram program];
    shaderProgram.vertexFunctionName = @"vertex_func";
    shaderProgram.fragmentFunctionName = @"fragment_func";
    geometry.firstMaterial.program = shaderProgram;
    
    return geometry;
}

```
至此多边形就算是创建完成了,这里面需要注意的是,最后我们加入了vertex_func和fragment_func两个文件,这里后面要用到,因为我们在没有做上色功能之前,需要自己给点云添加颜色,模拟rviz.


#### 定点着色器,片源着色器

```cpp
#include <metal_stdlib>
#include <SceneKit/scn_metal>

using namespace metal;

struct NodeBuffer {
    float4x4 modelTransform;
    float4x4 modelViewProjectionTransform;
    float4x4 modelViewTransform;
    float4x4 normalTransform;
    float2x3 boundingBox;
};

struct VertexInput {
    float3 position [[attribute(SCNVertexSemanticPosition)]];
    // 因为没有用到外部的颜色数据,所以,这块就先不用了
//    float3 color [[attribute(SCNVertexSemanticColor)]];
};

struct VertexOut {
    float4 position [[position]];
    float size [[point_size]];
    float4 color;
};


// 顶点着色器处理
vertex VertexOut vertex_func(VertexInput vertices [[stage_in]],
                             constant SCNSceneBuffer& scn_frame [[buffer(0)]],
                             constant NodeBuffer& scn_node [[buffer(1)]]) {
    VertexOut outVertex;
    // 设置每个顶点的位置
    outVertex.position = scn_node.modelViewProjectionTransform * float4(vertices.position, 1.0);
    // 设置每个顶点的大小
    outVertex.size = 1;
    // 设置每个顶点的颜色
    // 这里我们选择用一个简单的方式来进行上色填充,你也可以自己定义.
    // 主要目的是模拟rviz,让点云数据颜色看起来有一些变化.
    outVertex.color = float4(1-sin(vertices.position[2]),1-cos(vertices.position[2]),sin(vertices.position[2]),1);
    // 这里因为没有做点云上色,所以先不用呢.
//    outVertex.color = float4(vertices.color, 1);
    return outVertex;
}


// 片源着色器处理
fragment float4 fragment_func(VertexOut vert [[stage_in]]) {
// 不用做啥处理,直接透传
    return vert.color;
}
```
好了,跑起来看下,点云数据应该可以正常显示.


### 如何自定义控制相机

我们在雷达扫描环境的时候,通过手势来控制旋转相机控制查看场景,包括不限于,移动,旋转,缩放等

首先我们要先关闭自带的相机控制

```objc
//self = SCNView对象
self.allowsCameraControl = NO;
self.userInteractionEnabled = NO;
```

然后我们要加入自己的2个手势
```objc
// 拖动
UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(onPan:)];
pan.delegate = self;
[self addGestureRecognizer:pan];
        
// 捏合
UIPinchGestureRecognizer *pinch = [[UIPinchGestureRecognizer alloc] initWithTarget:self action:@selector(onPinch:)];
pinch.delegate = self;
[self addGestureRecognizer:pinch];

// 让多手势共存,我们要设置下delegate
#pragma mark - UIGestureRecognizerDelegate
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer {
    return YES;
}
```

下面处理2个手势的消息
#### *首先是捏合,用来做缩放*

```objc
// 最大缩放上限
static CGFloat const WORLD_SCALE_MAX = 5.0f;      
// 最大缩放下限         
static CGFloat const WORLD_SCALE_MIN = 0.1f;                

// 记录上次缩放的scale
@property (assign, nonatomic) CGFloat lastScale;


- (void)onPinch:(UIPinchGestureRecognizer *)sender {
    // 限制为必须2指有效
    if (sender.numberOfTouches == 2) {
        [self handlePinchScaleWithGestureRecognizer:sender];
    }
}

- (void)handlePinchScaleWithGestureRecognizer:(UIPinchGestureRecognizer *)sender
{
    if (sender.state == UIGestureRecognizerStateBegan) {
        self.lastScale = self.worldNode.scale.x;
    }
    //缩放
    CGFloat fix = 0;
    fix = sender.scale * self.lastScale;
    self.worldNode.scale = SCNVector3Make(fix, fix, fix);
    if (self.worldNode.scale.x <= WORLD_SCALE_MIN) {
        self.worldNode.scale = SCNVector3Make(WORLD_SCALE_MIN, WORLD_SCALE_MIN, WORLD_SCALE_MIN);
    }
    if (self.worldNode.scale.x >= WORLD_SCALE_MAX) {
        self.worldNode.scale = SCNVector3Make(WORLD_SCALE_MAX, WORLD_SCALE_MAX, WORLD_SCALE_MAX);
    }
}
```
> 这里大家已经看出来了,我缩放的控制实际是对整个场景进行了缩放.

#### *拖动,用来做移动和旋转*
其中包含单指和双指操作

```objc
/// 是否锁定单双指操作,产品需要,即在单指和双指出现时,只锁定第一次出现的
@property (assign, nonatomic) NSUInteger lockPanTouchNum;


- (void)onPan:(UIPanGestureRecognizer*)sender {
    if (sender.state == UIGestureRecognizerStateBegan) {
        self.lockPanTouchNum = sender.numberOfTouches;
    }
    if (sender.state == UIGestureRecognizerStateEnded ||
        sender.state == UIGestureRecognizerStateCancelled) {
        self.lockPanTouchNum = 0;
    }
    
    if (sender.numberOfTouches == 1 && self.lockPanTouchNum == sender.numberOfTouches) {
        // 单指旋转
        [self handleOneTouchToRotationWithGestureRecognizer:sender];
    } else if (sender.numberOfTouches == 2 && self.lockPanTouchNum == sender.numberOfTouches) {
        // 双指移动
        [self handleTouchToMoveWithGestureRecognizer:sender];
    }
    // 这个咱们不用管,这个是旋转时,是否要出现坐标轴模型的判断
    [self checkShowAxisIfNeedWithGestureRecognizer:sender];
}
```

##### 处理旋转

```objc
- (void)handleOneTouchToRotationWithGestureRecognizer:(UIPanGestureRecognizer *)sender
{
    CGPoint translation = [sender velocityInView:sender.view];
    // 水平旋转
    {
        SCNVector4 orientation = self.worldNode.orientation;
        GLKQuaternion glQuaternion = GLKQuaternionMake(orientation.x,
                                                       orientation.y,
                                                       orientation.z,
                                                       orientation.w);
        GLKQuaternion zMultiplier = GLKQuaternionMakeWithAngleAndAxis(translation.x/10000.0,
                                                                      0,
                                                                      0,
                                                                      1);
        glQuaternion = GLKQuaternionMultiply(glQuaternion, zMultiplier);
        
        self.worldNode.orientation = SCNVector4Make(glQuaternion.x,
                                                    glQuaternion.y,
                                                    glQuaternion.z,
                                                    glQuaternion.w);
    }
    
    //仰角旋转
    {
        SCNVector4 orientation = self.worldPerspectiveNode.orientation;
        GLKQuaternion glQuaternion = GLKQuaternionMake(orientation.x,
                                                       orientation.y,
                                                       orientation.z,
                                                       orientation.w);
        GLKQuaternion xMultiplier = GLKQuaternionMakeWithAngleAndAxis(translation.y/10000.0,
                                                                      1,
                                                                      0,
                                                                      0);
        glQuaternion = GLKQuaternionMultiply(glQuaternion, xMultiplier);
        
        self.worldPerspectiveNode.orientation = SCNVector4Make(glQuaternion.x,
                                                               glQuaternion.y,
                                                               glQuaternion.z,
                                                               glQuaternion.w);
        // 这里做一个仰角的限制,最低只能平视,最高到俯视
        if (self.worldPerspectiveNode.eulerAngles.x < -M_PI_2 ||
            self.worldPerspectiveNode.eulerAngles.x > 0)
        {
            self.worldPerspectiveNode.orientation = orientation;
        }
    }
    
    // 同步切换,这个不用管,不是重点
    [self.axisView changedOrientationWithWorldNode:self.worldNode worldPerspectiveNode:self.worldPerspectiveNode];
}
```

##### 仰角节点说明
```objc
// 设置仰角用的场景根节点
self.worldPerspectiveNode = [SCNNode node];
[self.scene.rootNode addChildNode:self.worldPerspectiveNode];
self.worldNode = [SCNNode node];
[self.worldPerspectiveNode addChildNode:self.worldNode];
```
> 因为仰角比较特殊,这里我们用一个专门的根节点来控制,和场景节点的关系如上所示
我们的其他模型都是挂载在self.worldNode上.


##### 处理双指移动

```objc
// 相机距离中心点的默认距离
static CGFloat const CAMERA_TO_ORIGIN_DISTANCE = 30.0f;    

/// 记录上次拖动的起始位置
@property (assign, nonatomic) SCNVector3 lastPosition;
@property (assign, nonatomic) SCNVector3 lastStartPosition;

- (void)handleTouchToMoveWithGestureRecognizer:(UIPanGestureRecognizer *)sender {
    if (sender.state == UIGestureRecognizerStateBegan) {
        self.lastPosition = self.worldNode.position;
        CGPoint translation = [sender translationInView:sender.view];
        self.lastStartPosition = [self convertToScenOfPoint:translation];
    }
    CGPoint translation = [sender translationInView:sender.view];
    SCNVector3 vex3 = [self convertToScenOfPoint:translation];
    vex3 = SCNVector3Make(vex3.x-self.lastStartPosition.x,
                          vex3.y-self.lastStartPosition.y,
                          vex3.z-self.lastStartPosition.z);
    SCNVector3 fix = SCNVector3Make(self.lastPosition.x + vex3.x,
                                    self.lastPosition.y + vex3.y,
                                    0);
    self.worldNode.position = fix;
}

- (SCNVector3)convertToScenOfPoint:(CGPoint)point
{
    CGFloat zFar = CAMERA_TO_ORIGIN_DISTANCE;
    double bigY = tan((double)(self.cameraNode.camera.fieldOfView/180/2)*M_PI) * (double)(zFar);
    double bigX = tan((double)(self.cameraNode.camera.fieldOfView/2/180)*M_PI) * (double)(zFar) * (double)(self.bounds.size.width/self.bounds.size.height);
    CGFloat alphaX = 2 * bigX / self.bounds.size.width;
    CGFloat alphaY = 2 * bigY / self.bounds.size.height;
    float x = -(CGFloat)bigX + point.x * alphaX;
    float y = (CGFloat)bigY - point.y * alphaY;
    SCNVector3 target = SCNVector3Make((float)(x), (float)(y), (float)(-zFar));
    SCNVector3 convertPoint = [self.cameraNode convertVector:target toNode:self.scene.rootNode];
//    NSLog(@"convertPoint X: %f,Y:%f，Z:%f", convertPoint.x, convertPoint.y, convertPoint.z);
    return convertPoint;
}

```

好了目前我们已经实现了双指移动,和单指旋转的逻辑,其中比较麻烦的是,拖动时的手势与模型移动旋转的同步问题.这里面查了一些资料,后面有时间再做详细说明.


### 未完可能待续