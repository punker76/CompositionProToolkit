<img src="https://cloud.githubusercontent.com/assets/7021835/16889814/1784ed78-4a9e-11e6-80d0-7c2084d6c960.png" alt="CompositionProToolkit"></img>

# Table of Contents

- [CompositionProToolkit Internals](#compositionprotoolkit-internals)
  - [Creating custom shaped `Visual` using `CanvasGeometry`]()
  - [Creating Masked Backdrop Brush using `ICompositionMask`]()
- [CompositionProToolkit Controls]()
  - [FluidProgressRing](#fluidprogressring)
  - [FluidWrapPanel](#fluidwrappanel)

**CompositionProToolkit** is a collection of helper classes for Windows.UI.Composition. The main difference between **CompositionProToolkit** and [**CompositionExpressionToolkit**](https://github.com/ratishphilip/CompositionExpressionToolkit) (_another project of mine_) is that **CompositionProToolkit** has a dependency on **Win2D** library.

# Installing from NuGet

To install **CompositionProToolkit**, run the following command in the **Package Manager Console**

```
Install-Package CompositionProToolkit
```

More details available [here](https://www.nuget.org/packages/CompositionProToolkit/).

# CompositionProToolkit Internals

## 1. Creating custom shaped `Visual` using `CanvasGeometry`
As of now, you can customize the shape of Visuals by applying a mask on the Visual. The mask is defined using a **CompositionMaskBrush**. In the **CompositionMaskBrush** the `Mask` is defined by a **CompositionSurfaceBrush**. Into the **CompositionSurfaceBrush** an image, which defines the mask, is loaded. In this image, the areas which are to masked in the Visual are transparent whereas the areas to be shown in the Visual are white.  

Using **CompositionProToolkit** you can now define a mask for the **Visual** using **Win2D**'s **CanvasGeometry**. It provides two interfaces **ICompositionMask** and **ICompositionMaskGenerator** and a static class **CompositionMaskFactory** which provides an object implementing the **ICompositionMaskGenerator**. Using the **ICompositionMaskGenerator** an object implementing the **ICompositionMask** can be created. This object represents the mask that needs to be applied to the Visual using a **CompositionMaskBrush**.

### Example

The following code

```C#
// Get the Generator
ICompositionMaskGenerator generator = CompositionMaskFactory.GetCompositionMaskGenerator(compositor);

//Create the visual
SpriteVisual visual = compositor.CreateSpriteVisual();
visual.Size = new Vector2(400, 400);
visual.Offset = new Vector3(200, 0, 0);

// Create the combined geometry
var ellipse1 = CanvasGeometry.CreateEllipse(generator.Device, 200, 200, 150, 75);
var ellipse2 = CanvasGeometry.CreateEllipse(generator.Device, 200, 200, 75, 150);
var combinedGeometry = ellipse1.CombineWith(ellipse2, Matrix3x2.Identity, CanvasGeometryCombine.Union);

// Create the CompositionMask
ICompositionMask compositionMask = await generator.CreateMaskAsync(visual.Size.ToSize(), combinedGeometry);

// Create SurfaceBrush from CompositionMask
var mask = compositor.CreateSurfaceBrush(compositionMask.Surface);
var source = compositor.CreateColorBrush(Colors.Blue);

// Create mask brush
var maskBrush = compositor.CreateMaskBrush();
maskBrush.Mask = mask;
maskBrush.Source = source;

visual.Brush = maskBrush;
```

creates the following output.  

<img src="https://cloud.githubusercontent.com/assets/7021835/15728977/0f9f397a-2815-11e6-9df2-65b9ad1f5e9f.PNG" />

You can also provide a fill color while creating a **ICompositionMask**. You can then use this **ICompositionMask** to create a **CompositionSurfaceBrush** which will achieve the same result. The previous example can also be written as

```C#
// Get the Generator
ICompositionMaskGenerator generator = CompositionMaskFactory.GetCompositionMaskGenerator(compositor);

//Create the visual
SpriteVisual visual = compositor.CreateSpriteVisual();
visual.Size = new Vector2(400, 400);
visual.Offset = new Vector3(200, 0, 0);

// Create the combined geometry
var ellipse1 = CanvasGeometry.CreateEllipse(generator.Device, 200, 200, 150, 75);
var ellipse2 = CanvasGeometry.CreateEllipse(generator.Device, 200, 200, 75, 150);
var combinedGeometry = ellipse1.CombineWith(ellipse2, Matrix3x2.Identity, CanvasGeometryCombine.Union);

// Create the CompositionMask
ICompositionMask compositionMask = await generator.CreateMaskAsync(visual.Size.ToSize(), combinedGeometry, Colors.Blue);

// Create SurfaceBrush from CompositionMask
var surfaceBrush = compositor.CreateSurfaceBrush(compositionMask.Surface);

visual.Brush = surfaceBrush;
```


**ICompositionMask** provides a **RedrawAsync** API which allows you to update the geometry of the mask (and thus the shape of the Visual).  

Here is an example of a **CanvasAnimatedControl** having two visuals - A blue rectangular visual in the background and a red visual in the foreground. The red visual's mask is redrawn periodically to give an impression of animation. (_see the **Sample** project for more details on how it is implemented_)

```C#
private async void AnimatedCanvasCtrl_OnDraw(ICanvasAnimatedControl sender, CanvasAnimatedDrawEventArgs args)
{
    angle = (float)((angle + 1) % 360);
    var radians = (float)((angle * Math.PI) / 180);
    
    // Calculate the new geometry based on the angle
    var updatedGeometry = outerGeometry.CombineWith(combinedGeometry, 
                                Matrix3x2.CreateRotation(radians, new Vector2(200, 200)),
                                CanvasGeometryCombine.Exclude);
        
    // Update the geometry in the Composition Mask
    await animatedCompositionMask.RedrawAsync(updatedGeometry);
}
```

<img src="https://cloud.githubusercontent.com/assets/7021835/15728986/1baeab9c-2815-11e6-8e93-846b70a2a3ea.gif" />

## 2. Creating Masked Backdrop Brush using `ICompositionMask`
**CompositionProToolkit** now provides the following extension method for **Compositor** to create a masked Backdrop brush.

```C#
public static CompositionEffectBrush CreateMaskedBackdropBrush(this Compositor compositor, ICompositionMask mask,
            Color blendColor, float blurAmount, CompositionBackdropBrush backdropBrush = null)
```  

Using this method, you can apply a BackdropBrush with a custom shape to a visual. You can provide a **Color** to blend with the BackdropBrush, the amount by which the BackdropBrush should be blurred and an optional **CompositionBackdropBrush**. If no **CompositionBackdropBrush** is provided by the user then this method creates one.

**Note** : _Create only one instance of **CompositionBackdropBrush** and reuse it within your application. It provides a better performance._

<img src="https://cloud.githubusercontent.com/assets/7021835/16091854/562d255c-32ea-11e6-8952-424a513741ea.gif" />

# CompositionProToolkit Controls

## 1. FluidProgressRing
**FluidProgressRing** is a concept design for a modern version of the **ProgressRing** control in UWP.The **ProgressRing** control has remained the same since Windows 8 and it is high time it got a refresh. The **FluidProgressRing** consists of a set of small circles (**nodes**) rotating about a common center. Each node rotates until it hits the adjacent node (then it slows down to a stop). The adjacent node then rotates and hits the next node and this process continues. The animations are done using the **Windows.UI.Composition APIs** and provide a smooth look and feel.

<img src="https://cloud.githubusercontent.com/assets/7021835/16522118/838f2eec-3f50-11e6-825c-20e07300339c.gif" />

To use it you can just add it to your XAML and it will start rotating. It has a default size of **70 x 70** and has **7** nodes. The good thing about the **FluidProgressRing** control is that it can be easily configured. Here are the dependency properties which you can modify to alter the **FluidProgressRing's** appearance

| Dependency Property | Type | Description | Default Value |  
|----|----|----|----|
| **MaxNodes** | `int` | The maximum number of nodes that can be accommodated within the FluidProgressRing | **7** |   
| **ActiveNodes** | `int` | The number of stationary nodes in the FluidProgressRing. The FluidProgressRing will have an additional node which will be in motion. ActiveNodes should be less than or equal to MaxNodes. | **6** |  
| **NodeDuration** | `TimeSpan` | The time it takes for a node to travel and hit the adjacent node. | **0.5s** |  
| **RingDuration** | `TimeSpan` | The duration for one complete rotation of the FluidProgressRing. | **5s** |  
| **NodeSizeFactor** | `double` | The ratio of the node diameter to the width of the FluidProgressRing | **0.15** |  
| **NodeColor** | `Color` | The color of the nodes | **Blue** |  

You can also change the **Width** and **Height** properties of **FluidProgressRing** control. The final size of the **FluidProgressRing** control will be the largest square that can fit in the defined **Width** and **Height**.

## 2. FluidWrapPanel

<img src="https://cloud.githubusercontent.com/assets/7021835/16889802/077dc724-4a9e-11e6-8475-7693138f0b39.gif" alt="FluidWrapPanel demo"></img>
