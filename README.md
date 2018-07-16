# Using Vision in Real Time with ARKit

# 在ARKit中实时使用Vision

Manage Vision resources for efficient execution of a Core ML image classifier, and use SpriteKit to display image classifier output in AR.

管理Vision资源来有效执行Core ML图片分类器,交使用SpriteKit来在AR中展示图片分类器的输出结果.

## Overview 总览

This sample app runs an [ARKit][0] world-tracking session with content displayed in a SpriteKit view. The app uses the [Vision][1] framework to pass camera images to a [Core ML][2] classifier model, displaying a label in the corner of the screen to indicate whether the classifier recognizes anything in view of the camera. After the classifier produces a label for the image, the user can tap the screen to place that text in AR world space. 

该示例app运行一个[ARKit][0] world-tracking session,并用一个SpriteKit视图来展示内容.还使用了[Vision][1]框架来将相机画面传递给一个 [Core ML][2]分类模型中,然后在屏幕的左上角有一个label会展示图片分类器识别出的物体.在分类器给图片产生一个标签后,用户可以点击屏幕来放置一个AR文本.

- Note: The Core ML image classifier model doesn't recognize and locate the 3D positions of objects. (In fact, the `Inceptionv3` model attempts only to identify an entire scene.) When the user taps the screen, the app adds a label at a real-world position corresponding to the tapped point. How closely a label appears to relate to the object it names depends on where the user taps.
- 注意:Core ML图片分类模型并不能识别也不能定位物体的3D位置.(事实上,`Inceptionv3`模型只会尝试对整个场景做个标志.)当用户点击屏幕,app会根据点击的位置,在真实世界的对应位置上,添加一个label.这个label距离物体有多近,取决于用户点击的位置.


[0]:https://developer.apple.com/documentation/arkit
[1]:https://developer.apple.com/documentation/vision
[2]:https://developer.apple.com/documentation/coreml

## Getting Started 开始

ARKit requires iOS 11.0 and a device with an A9 (or later) processor. ARKit is not available in iOS Simulator. Building the sample code requires Xcode 9.0 or later.

ARKit 要求 iOS 11.0 及一个 A9 (或以上) 处理的设备. ARKit在iOS模拟器上不可用. 运行救命代码需要 Xcode 9.0 及以上.

## Implement the Vision/Core ML Image Classifier 实现Vision/Core ML图片分类器



The sample code's [`classificationRequest`](x-source-tag://ClassificationRequest) property, [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method, and [`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) method manage:

示例代码中的[`classificationRequest`](x-source-tag://ClassificationRequest)属性,[`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage)方法,及[`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) 方法:

- A Core ML image-classifier model, loaded from an `mlmodel` file bundled with the app using the Swift API that Core ML generates for the model

  一个Core ML图片分类器模型,从`mlmodel`文件中加载,使用了Core ML框架为该模型创建的Swift API.

- [`VNCoreMLRequest`][3] and [`VNImageRequestHandler`][4] objects for passing image data to the model for evaluation

  [`VNCoreMLRequest`][3] 和 [`VNImageRequestHandler`][4] 对象用来将图片数据传递到模型以供处理.

For more details on using [`VNImageRequestHandler`][4], [`VNCoreMLRequest`][3], and image classifier models, see the [Classifying Images with Vision and Core ML][5] sample-code project. 

想知道更多关于 [`VNImageRequestHandler`][4], [`VNCoreMLRequest`][3]和图片分类器模型的信息,参见[Classifying Images with Vision and Core ML][5] 示例代码.

[3]:https://developer.apple.com/documentation/vision/vncoremlrequest
[4]:https://developer.apple.com/documentation/vision/vnimagerequesthandler
[5]:https://developer.apple.com/documentation/vision/classifying_images_with_vision_and_core_ml

## Run the AR Session and Process Camera Images 运行AR Session及处理相机图像

The sample `ViewController` class manages the AR session and displays AR overlay content in a SpriteKit view. ARKit captures video frames from the camera and provides them to the view controller in the [`session(_:didUpdate:)`][6] method, which then calls the [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method to run the Vision image classifier.

示例中的`ViewController`类管理着AR session,并在SpriteKit视图中展示AR内容层.ARKit从相机中捕捉视频帧,并在 [`session(_:didUpdate:)`][6] 方法中提供给控制器,然后调用[`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage)方法来运行Vision图片分类器.

``` swift
func session(_ session: ARSession, didUpdate frame: ARFrame) {
    // Do not enqueue other buffers for processing while another Vision task is still running.
    // 当另一个Vision任务正在运行时,不要将新的buffers添加到队列中
    // The camera stream has only a finite amount of buffers available; holding too many buffers for analysis would starve the camera.
    // 相机stream只有一个有限数量的buffers;持有太多buffers来分析,将会卡住相机.
    guard currentBuffer == nil, case .normal = frame.camera.trackingState else {
        return
    }
    
    // Retain the image buffer for Vision processing.
    // 强引用图片buffer以供Vision处理.
    self.currentBuffer = frame.capturedImage
    classifyCurrentImage()
}
```
[View in Source](x-source-tag://ConsumeARFrames)

[6]:https://developer.apple.com/documentation/arkit/arsessiondelegate/2865611-session

## Serialize Image Processing for Real-Time Performance 连续图片处理的实时性能表现

The [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method uses the view controller's `currentBuffer` property to track whether Vision is currently processing an image before starting another Vision task. 

 [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) 方法使用控制器的`currentBuffer`属性来判断Vision是否正在处理一幅画面,然后再决定是否开启另一个Vision任务.

``` swift
// Most computer vision tasks are not rotation agnostic so it is important to pass in the orientation of the image with respect to device.
// 大部分计算型vision任务需要明确知道图片的朝向信息,所以需要将当前图片的朝向传入.
let orientation = CGImagePropertyOrientation(UIDevice.current.orientation)

let requestHandler = VNImageRequestHandler(cvPixelBuffer: currentBuffer!, orientation: orientation)
visionQueue.async {
    do {
        // Release the pixel buffer when done, allowing the next buffer to be processed.
        // 处理完成后释放pixel buffer,允许处理下一个buffer.
        defer { self.currentBuffer = nil }
        try requestHandler.perform([self.classificationRequest])
    } catch {
        print("Error: Vision request failed with error \"\(error)\"")
    }
}
```
[View in Source](x-source-tag://ClassifyCurrentImage)

- Important: Making sure only one buffer is being processed at a time ensures good performance. The camera recycles a finite pool of pixel buffers, so retaining too many buffers for processing could starve the camera and shut down the capture session. Passing multiple buffers to Vision for processing would slow down processing of each image, adding latency and reducing the amount of CPU and GPU overhead for rendering AR visualizations.

  重要提示:确保只有一个buffer正在处理中可以有效保证良好的性能表现.相机是循环利用一个有限数量的pixel buffer池,所以持有太多的buffer会卡住相机,并关闭capture session.同时传递多个buffers给Vision处理会拖慢每帧的处理速度,增加延迟,并减少CPU和GPU渲染AR场景的开销.

In addition, the sample app enables the [`usesCPUOnly`][7] setting for its Vision request, freeing the GPU for use in rendering.

另外,示例代码在Vision请求中启用了 [`usesCPUOnly`][7] ,释放了GPU以供渲染.

[7]:https://developer.apple.com/documentation/vision/vnrequest/2923480-usescpuonly

## Visualize Results in AR 在AR中呈现结果

The [`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) method stores the best-match result label produced by the image classifier and displays it in the corner of the screen. The user can then tap in the AR scene to place that label at a real-world position. Placing a label requires two main steps. 

 [`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) 方法存储了图片分类器中产生的最佳匹配标签,并显示在左上角的屏幕上.用户可以点击AR场景来在真实世界中放置这个label.放置一个label需要两大步骤.

First, a tap gesture recognizer fires the [`placeLabelAtLocation(sender:)`](x-source-tag://PlaceLabelAtLocation) action. This method uses the ARKit [`hitTest(_:types:)`][8] method to estimate the 3D real-world position corresponding to the tap, and adds an anchor to the AR session at that position.

首先,点击手势识别器触发 [`placeLabelAtLocation(sender:)`](x-source-tag://PlaceLabelAtLocation) .这个方法使用了ARKit中的 [`hitTest(_:types:)`][8] 方法来根据屏幕点击估计3D真实世界的位置,并在该位置处向AR session添加一个锚点.

``` swift
@IBAction func placeLabelAtLocation(sender: UITapGestureRecognizer) {
    let hitLocationInView = sender.location(in: sceneView)
    let hitTestResults = sceneView.hitTest(hitLocationInView, types: [.featurePoint, .estimatedHorizontalPlane])
    if let result = hitTestResults.first {
        
        // Add a new anchor at the tap location.
        // 在点击位置添加一个新的锚点.
        let anchor = ARAnchor(transform: result.worldTransform)
        sceneView.session.add(anchor: anchor)
        
        // Track anchor ID to associate text with the anchor after ARKit creates a corresponding SKNode.
        // 将锚点ID与文本关联起来,当ARKit创建对应的SKNode之后就可根据锚点ID获取关联的文本.
        anchorLabels[anchor.identifier] = identifierString
    }
}
```
[View in Source](x-source-tag://PlaceLabelAtLocation)

Next, after ARKit automatically creates a SpriteKit node for the newly added anchor, the [`view(_:didAdd:for:)`][9] delegate method provides content for that node. In this case, the sample [`TemplateLabelNode`](x-source-tag://TemplateLabelNode) class creates a styled text label using the string provided by the image classifier.

下一步,在ARKit自动给新添加的锚点创建SpriteKit节点后,[`view(_:didAdd:for:)`][9] 代理方法会给该节点提供内容.在本例中,示例[`TemplateLabelNode`](x-source-tag://TemplateLabelNode) 类根据图片分类器提供的字符串创建了一个文本label.

``` swift
func view(_ view: ARSKView, didAdd node: SKNode, for anchor: ARAnchor) {
    guard let labelText = anchorLabels[anchor.identifier] else {
        fatalError("missing expected associated label for anchor")
    }
    let label = TemplateLabelNode(text: labelText)
    node.addChild(label)
}
```
[View in Source](x-source-tag://UpdateARContent)

[8]:https://developer.apple.com/documentation/arkit/arskview/2875733-hittest
[9]:https://developer.apple.com/documentation/arkit/arskviewdelegate/2865588-view