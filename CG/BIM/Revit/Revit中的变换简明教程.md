
虽然 Revit 变换在初看起来可能是一个复杂的主题，但它们专注于简单且基础的几何概念，通过编程方式来实现。本质上，它们是用于根据预定义逻辑在 3D 空间中重新定位点或对象的方法。
#### 平移  

平移涉及使用向量移动一个点。该点基于向量的方向和长度进行移动。

![[Pasted image 20251105112518.png]]


```csharp
public class Transformations
{
    public void MakeTranslation()
    {
        XYZ pointA = new XYZ(10, 10, 0);
        XYZ translationVector = new XYZ(5,10,0);
        
        Transform transform_Translation = Transform.CreateTranslation(translationVector);

        XYZ pointA_Translated = transform_Translation.OfPoint(pointA);
        // pointA_Translated 坐标将是：15,20,0
    }
}
```


#### 旋转  

旋转涉及沿着定义的圆形路径移动一个点，并按照旋转角度进行。

##### 围绕坐标系原点的旋转

![[Pasted image 20251105112814.png]]

```csharp
public void MakeRotation()
{
    XYZ pointA = new XYZ(10, 10, 0);
    XYZ rotationAxis = new XYZ(0, 0, 1);

    double rotationAngle = 1.5707963268;

    // 角度单位是弧度，对应 90 度。
    Transform transform_Rotation = Transform.CreateRotation(rotationAxis, rotationAngle);

    XYZ pointA_Rotated = transform_Rotation.OfPoint(pointA);
    // pointA_Rotated 坐标将是：-10,10,0
}
```

##### 围绕特定点的旋转

![[Pasted image 20251105113002.png]]

```csharp
public static void MakeRotationAtPoint()
{
    XYZ pointA = new XYZ(30,30,0);
    XYZ pointO = new XYZ(20,20,0);
    XYZ rotationAxis = new XYZ(0, 0, 1);

    double rotationAngle = 1.5707963268;
    Transform transform_Rotation = Transform.CreateRotationAtPoint(rotationAxis, rotationAngle, pointO);

    XYZ pointA_Rotated = transform_Rotation.OfPoint(pointA);
    // pointA_Rotated 坐标将是：10,30,0
}
```

  
#### 平移 + 旋转

  描述多个变换的 Transform 对象在处理某些 Revit 对象时相当常见。
  
  例如，每个 FamilyInstance 对象都包含一个 Transform 对象，该对象通知我们有关空间中的族实例位置。含义 ：从坐标系的原点开始，它已经平移了多远，以及它相对于坐标系的轴旋转到哪个角度。
  
![[Pasted image 20251105113115.png]]

```csharp
public static void MakeRotationAtPointPlusTranslation()
{
    XYZ pointA = new XYZ(30, 30, 0);
    XYZ pointO = new XYZ(20, 20, 0);
    XYZ rotationAxis = new XYZ(0, 0, 1);
    double rotationAngle = 1.5707963268;

    Transform transform_Rotation = Transform.CreateRotationAtPoint(rotationAxis, rotationAngle, pointO);
    XYZ translationVector = new XYZ(5, 10, 0);

    Transform transform_Translation = Transform.CreateTranslation(translationVector);
    
    Transform transform_RotationPlusTranslation = transform_Translation.Multiply(transform_Rotation);

    XYZ pointA_RotatedAndTranslated = transform_RotationPlusTranslation.OfPoint(pointA);

    // pointA_RotatedAndTranslated 坐标是 15,40,0
}
```

#### 结论  

正如我们所见，Revit 变换是相当简单的概念。使用它们将在处理 Revit 对象时为您提供更大的灵活性，并更好地理解全局坐标系和对象局部坐标系之间的转换。