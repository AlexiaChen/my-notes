
我在使用 Revit API 时遇到的挑战之一是定义模型元素相对于彼此的空间位置。在自动控制墙内的开口时，我必须考虑这个问题，以确保它们的位置符合特定的结构约束。

默认情况下，Revit API 返回与内部原点相关的点坐标，该点被视为模型全局坐标系的中心。但是，在我的情况下，我希望洞口的坐标与特定于墙的坐标系相关。这将使结构验证算法更易于设计和编程。

下面介绍了我开发并一直使用的一种方法，用于将坐标从模型坐标系转换为特定于元素的坐标系。我将首先介绍此方法的几何运算步骤，然后提供相关的 C# 代码。

## 坐标转换

![[Pasted image 20251105134404.png]]
我们将使用的示例是一堵简单的墙。我们使用该墙的下角之一 **PointO** 作为原点，全局坐标为 **（2， 4， 0）** 建立该墙的局部坐标系。局部坐标系的轴与此时相交的墙的三个边对齐。

**点 A** 是水平线与墙面之一相交的位置，全局（模型）坐标为 **（5， 6， 2），** 局部（墙）坐标为 **（3.60 ，0 ，2）。** 我们的目的是了解如何将  **PointA** 的 坐标从全局坐标系转换为局部坐标系。

![[Pasted image 20251105134730.png]]

我们首先定义一个从模型的内部原点开始并向 **O 点**延伸的向量。然后我们定义与此向量相反的向量，并使用它来平移 **PointA**。

![[Pasted image 20251105134749.png]]

我们计算**α**：墙的**局部 x 轴**与模型的**全局 x 轴**之间的角度。

![[Pasted image 20251105134815.png]]

我们将平移的**点 A** 绕模型的内部原点旋转角度**α**。结果是一个全局（模型）坐标为 **（3.60， 0， 2）** 的点。我们注意到，当与局部（墙）坐标系相关时，这些坐标实际上与 **PointA** 的坐标匹配。


总之，该过程涉及定义与元素相关的局部坐标系，并使用变换将全局坐标转换为局部坐标。即使我们可以自由定义局部坐标系的原点及其轴的方向，转换坐标的逻辑也保持一致。

此过程可以通过以下 C# 代码以编程方式表达：

```csharp
public class CoordinatesConverter
    {
        public void ConvertCoordinates()
        {
            XYZ pointA = new XYZ(5, 6, 2);
            XYZ pointB = new XYZ(8, 8, 0);
            XYZ pointO = new XYZ(2, 4, 0);
            XYZ rotationAxis = new XYZ(0, 0, 1);
            XYZ translatedAndRotatedPointA;
            XYZ translatedPointB;

            Transform transform_Translation = Transform.CreateTranslation(pointO);
            Transform transform_invertedTranslation = transform_Translation.Inverse;

            // To calculte alphaAngle we first need to translate pointB. The resulting vector will have the same direction as the wall's x-axis.  
            translatedPointB = transform_invertedTranslation.OfPoint(pointB);
            double alphaAngle = translatedPointB.AngleTo(new XYZ(1, 0, 0));

            // Since we want our rotation to be done clockwise, alphaAngle needs to be negative
            Transform transform_Rotation = Transform.CreateRotation(rotationAxis, alphaAngle * -1);
            Transform transform_TranslationPlusRotation = transform_Rotation.Multiply(transform_invertedTranslation);

            translatedAndRotatedPointA = transform_TranslationPlusRotation.OfPoint(pointA);
            // translatedAndRotatedPointA coordinates will be : 3.60, 0, 2
        }
    }
```

