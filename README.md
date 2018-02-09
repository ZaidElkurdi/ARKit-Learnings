
# ARKit Learnings
## Testing
I started out by building to my phone every time I wanted to test a change I made. However, the compilation and transfer time started to add up; even the simplest changes would take ~5 minutes to test. I'd strongly recommend doing as much testing as possible in an  Xcode playground because the iteration time is so much faster. The way I'm doing this is by setting up a SCNView and adding a camera:

    let sceneView = SCNView(frame: CGRect(x: 0, y: 0, width: 500, height: 500))
    sceneView.autoenablesDefaultLighting = true
    
    let scene = SCNScene()
    
    var cameraNode = SCNNode()
    cameraNode.camera = SCNCamera()
    cameraNode.position = SCNVector3(x: 0, y: 0, z: 15)
    scene.rootNode.addChildNode(cameraNode)
    
    sceneView.scene = scene
    PlaygroundPage.current.liveView = sceneView


## Shader Modifiers
I wanted to create something like this:

![Loading Spinner](https://4.bp.blogspot.com/-tqB3UP6KYaM/VItDu0F9kdI/AAAAAAAAATs/QB0X4AqWWY0/s1600/spinningwheel.gif)

This would have been super easy with a UIBezierPath, but I'm using SceneKit so things got a little more complicated. After much Googling, I settled on using a [Shader Modifier](https://developer.apple.com/documentation/scenekit/scnshadable). Shader Modifiers allow you to customize how SCNNodes are rendered and, conveniently, can also be used for custom animations. In order to achieve the growing/shrinking circle effect I wrote a little bit of GLSL code and tested it in a Playground:

    uniform float progress;
    #pragma transparent
    #pragma body
    vec4 mPos = u_inverseModelViewTransform * vec4( _surface.position, 1.0 );
    _surface.transparent.a = atan( mPos.x, mPos.z ) <= (2.0 * 3.141592 * progress - 3.141592) ? 1.0 : 0.0;

All this code is doing is taking the *atan* of the current vertex and deciding whether or not to show it based on the current progress. For 0% we only show vertices with *atan* <= -π (none) and, for 50% <= 0, and for 100% <= π (all of them). In order to animate this we can create a CABasicAnimation in the SCNNode subclass for the progress key path:

    let animation = CABasicAnimation(keyPath: "geometry.progress")
    animation.toValue = percentageCompleted
    animation.duration = duration
    addAnimation(animation, forKey: nil)

This worked fine in the Playground, but didn't work when I dropped it into my SCNNode subclass and also gave no indication of what went wrong. I knew that ARKit ran on Metal, but, according to the Apple docs, "SceneKit automatically translates GLSL syntax to Metal shader syntax". Eventually, I set a symbolic breakpoint in the MTLLibrary's initializer and saw what the code looked like when it was translated. The problem? GLSL's *atan* expects two arguments, and Metal's version expects one argument. I switched out *atan* for *atan2* and all was well:

    uniform float progress;
    #pragma transparent
    #pragma body
    vec4 mPos = u_inverseModelViewTransform * vec4( _surface.position, 1.0 );
    _surface.transparent.a = atan2( mPos.x, mPos.z ) <= (2.0 * 3.141592 * progress - 3.141592) ? 1.0 : 0.0;

   
## Scaling
I needed to use large nodes so that they could still be seen by the user when they were far away, but this meant that  when a node was only a couple meters away it would take up the user's whole screen. I solved this by scaling down the nodes to a maximum size when they got within a certain distance:

	let nodeScalingDistance = 10.0
    displayedNodes.forEach({ currNode in
      let distanceToNode = pointOfView.position.distance(to: currNode.position)
      let scale = min(1.0, max(0.1, distanceToNode / nodeScalingDistance))
      currNode.childNodes.forEach({ $0.scale = SCNVector3(x: scale, y: scale, z: scale) })
    })
At first I was only setting *currNode.scale*, but as it turns out I needed to set the scale of all the child nodes because the scale of the parent node didn't get passed down to its children automatically.

## Haptic Feedback
I wanted to add haptic feedback for certain events during the AR experience. I initially tried just adding a UISelectionFeedbackGenerator in the *ARSCNViewDelegate*'s *renderer(_ renderer: SCNSceneRenderer, updateAtTime: TimeInterval)* method. To my surprise this crashed whenever *selectionChanged()* was called and printed some cryptic internal library error message. Because the delegate method doesn't get called on the main thread  I was doing some UI changes in a dispatch_async block and also calling *selectionChanged()* in the block. As it turns out UIFeedbackGenerator isn't thread safe, so I pulled the call out of the block and the haptic feedback worked as expected.
