# ROS for iOS 的一些笔记

## 开始

之前工作中用到了ROS,这里简单做一个记录.

#### 相关资料

- [iOS版本库git代码地址](https://github.com/furushchev/ROSiOS)
- [ROS wiki地址](https://wiki.ros.org)

#### 通过pods导入ROS库


```c
  pod "ROSiOS"
  pod "ROSiOS-std_msgs"
  pod 'ROSiOS-geometry_msgs'
  pod 'ROSiOS-nav_msgs'
  pod 'ROSiOS-rosgraph_msgs'
  pod 'ROSiOS-sensor_msgs'
  pod 'ROSiOS-tf'
```

## 初始化准备工作

### *创建节点*
- 首先需要连接到机器人的WiFi中

> 你应该有办法来判断设备当前是否在机器人网络中.

- 配置环境变量<重要>

```objc
// ROS_MASTER_URI=http://localhost:11311
// 端口号默认为11311
NSString *master_uri = [@"ROS_MASTER_URI=" stringByAppendingString:self.masterURI];
// 获取本机在当前网络下的IP地址
NSString *ip = [self getMyIPAddress];
// ROS_HOSTNAME=xxx.xxx.xxx.xxx
NSString *hostname = [@"ROS_HOSTNAME=" stringByAppendingString:ip];
// 设置环境
putenv((char *)[master_uri UTF8String]);
putenv((char *)[hostname UTF8String]);
```

- 初始化节点

```objc
// nodeName 当相于唯一id,master可以用过这个标识来查看接入的节点
// 我们使用<自定义前缀>+<设备标识>的组合来当做Name
// 自定义前缀包含设备平台,如iOS,android
int argc = 0;
char **argv = NULL;
NSString *nodeName = [self clientNodeName];
ros::init(argc, argv, [nodeName UTF8String]);

// check master用来检查master节点是否有效
if (ros::master::check()) {
    NSLog(@"master is available")
} else {
    NSLog(@"error")
}
```

- 启动节点

```objc
ros::start();
```
> 实际上创建第一个 NodeHandle 会自动调用它,我这里在init之后,显性调用一次.看起来可能似乎更完整一些

### *如何关闭节点*

```objc
ros::shutdown();
```
> - 当你所有的NodeHandles销毁之后会自动调用,但是ROS允许你显性调用,这样看起来似乎比较明显
> - 这里需要注意的是shutdown时,你需要确保与机器人网络通畅,否则会超时.
> - 如果您在断开时没有网络,shutdown会阻塞当前线程.直到超时.

## 关于Publish
### *如何控制机器人移动*

- 如果您创建了节点并且master有效的话,此时就可以进行订阅广播等操作了.
- 先声明一个Publisher,用来专门发送Twist指令的

```objc
@interface YourManager () {
    ros::Publisher twistPublisher;
}
```

- 通知master我要给"/cmd_vel"发消息

```objc
ros::NodeHandle n;
const char *topic = [@"/cmd_vel" UTF8String];
self->twistPublisher = n.advertise<geometry_msgs::Twist>(topic, 1);
```

- 好了,现在您可以给master发送消息了,这里需要使用geometry_msgs::Twist类型的消息,它里面包含2个三维向量(geometry_msgs::Vector3),一个控制线速度的"linear",和控制角速度的"angular"

```objc
geometry_msgs::Twist twist;
twist.linear.x = linear;
twist.linear.y = 0;
twist.linear.z = 0;
twist.angular.x = 0;
twist.angular.y = 0;
twist.angular.z = angular;
self->twistPublisher.publish(twist);
```
> 这里需要注意的是,如果您想叫机器人一直移动,您需要使用循环来发送.

## 关于Subscribe

### *如何获取机器人位置信息*

- 我们使用订阅"\tf"信息来获取机器人的坐标
- 先声明一个ros::Subscriber

```objc
@interface YourManager () {
    ros::Subscriber tfSubscriber;
}
```
- 订阅topic

```objc
ros::NodeHandle n;
const char *topic = [@"\tf" UTF8String];
// tfCallBack是你的回调函数
self->tfSubscriber = n.subscribe(topic, 1, tfCallBack);
```
- 处理回调

```objc
// 回调函数
void tfCallBack(const tf::tfMessageConstPtr msg)
{
    geometry_msgs::TransformStamped transformStamped = msg->transforms[0];
    std_msgs::Header header = transformStamped.header;
    std::string frame_id = header.frame_id;
    std::string child_frame_id = transformStamped.child_frame_id;
    
    const char *odom = [@"\odom" UTF8String];
    const char *base_link = [@"\base_link" UTF8String];
    
    // 一些校验,过滤掉无用的数据
    if (strcmp(frame_id.c_str(), odom)==0 && strcmp(child_frame_id.c_str(), base_link)==0)
    {
        geometry_msgs::Transform transform = transformStamped.transform;
        // translation是机器人坐标
        geometry_msgs::Vector3 translation = transform.translation;
        // rotation是机器人的角度
        geometry_msgs::Quaternion rotation = transform.rotation;
        
        // 你的处理....
        
    }
}
```


- 关于Spin,这个是做订阅时一个重要的操作,否则无法收到回调.

```objc
ros::Rate loop_rate(10);
while (ros::ok() && <你的开关>) {
    // 这里我们通过循环来控制所以使用spinOnce,这样做比较灵活
    ros::spinOnce();
    loop_rate.sleep();
}
```

### *如何关闭订阅*

```objc
self->tfSubscriber.shutdown();
```

### *如何获取激光雷达点云数据*

- 与获取位置信息类似通过订阅广播来获取,以下略


## 关于Param

### *set*

```objc
ros::NodeHandle n;
n.setParam(key.UTF8String, value.UTF8String);
```

### *get*

```objc
ros::NodeHandle n;
if (!n.hasParam(key.UTF8String)) {
    std::string result;
    if (!n.getParam(key.UTF8String, result)) {
        // 你的操作...
        
    }
}

```


## 关于ClientService

### *一个简单的请求*

```objc
ros::NodeHandle n;
// service是你服务的标识
ros::ServiceClient ser_client = n.serviceClient<client::report>([service UTF8String]);

//这个是自己生成的结构,为了方便我们的结构中参数只传字符串,通过JSON来做参数.这样就不用生成大量头文件了.
client::report params;
params.request.request = [JSON UTF8String];
if (ser_client.call(params)) {
    NSString *response = [NSString stringWithUTF8String:params.response.response.c_str()];
    // 你的处理...
    
}
```


## 未完可能待续....