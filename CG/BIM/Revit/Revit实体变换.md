
使用 Revit API 时，我们经常执行涉及多个几何图形的任务。例如，我们可以比较它们的位置或确定它们的交叉点。但是，请务必注意，即使这些几何图形在 Revit 3D 空间中看起来很接近，它们也可能与不同的坐标系相关。在这种情况下，在引用它们各自的坐标时比较这些几何形状可能会给我们带来不正确的结果。

我曾经不得不编写一个脚本来检查链接模型中的 MEP 元素是否与主机模型中的 MEP 元素具有完全相同的位置。奇怪的是，即使 Revit 显示了所有要对齐的元素，脚本的结果也表达了相反的情况。我最终发现，每个模型相对于内部原点的元素定位不同，导致结果不正确。虽然，两个模型的元素与项目基点的位置相同，这解释了视觉对齐。

解决类似问题的一种方法是使用 Revit 变换并将其应用于几何图形。它们允许将几何的坐标从一个系统转换为另一个系统。除此之外，变换对于简单地在同一个 Revit 模型中移动几何图形也很有用，而无需考虑多个坐标系。

## 移动实体

  为了更好地了解如何在 Revit 中将变换应用于几何体，让我们考虑一个简单的示例：平移和旋转墙。我们首先定义一种方法，该方法将墙的几何体作为参数，对其应用所需的平移和旋转，然后返回生成的几何体。我们将此方法命名为TransformSolid

我们将使用 Revit API 提供的三个类：

- 变换 ：一个类，其中包含在 3D 空间中移动点或几何体的指令。
- 实体 ：表示在 Revit 中建模的任何对象的几何数据的类。
- SolidUtils ：执行与实体相关的多个作的类，其中使用静态方法 SolidUtils.CreateTransformed（） 应用转换。

虽然此示例涉及基本几何体，但相同的方法也可用于更复杂的几何体。

![[Pasted image 20251105113934.png]]

```csharp
public class Transformations
{
    public static Solid TransformSolid(Solid wallSolid)
    {

        XYZ translationVector = new XYZ(3, 5, 0);
        XYZ rotationAxis = new XYZ(0, 0, 1);
        XYZ pointO = new XYZ(8, 10, 0);

        double rotationAngle = 1.5707963268;

        Transform transform_Translation = Transform.CreateTranslation(translationVector);

        Transform transform_Rotation = Transform.CreateRotationAtPoint(rotationAxis, rotationAngle, pointO);

        Transform transform_TranslationPlusRotation = transform_Rotation.Multiply(transform_Translation);

        Solid transformedSolid = SolidUtils.CreateTransformed(wallSolid, transform_TranslationPlusRotation);

        return transformedSolid;
    }
}
```

## 转换坐标系

  到目前为止，我们已经了解了如何将变换应用于实体。这种相同的方法实际上用于在具有不同坐标系的两个不同的 Revit 模型之间转换几何的坐标。现在的问题是：我们如何找到进行此类转换所需的 Transform 对象。

在 Revit API 中，链接模型由两个类表示：RevitLinkType 和 RevitLinkInstance。后者实际上提供了一个方法来检索我们寻找的 Transform 对象：RevitLinkInstance.GetTransform（）。让我们将此对象标识为 linkModelTransform。

要将几何的坐标从链接模型的系统转换为主模型的系统：我们可以直接使用 **linkModelTransform** 作为 SolidUtils.CreateTransformed（） 方法的参数。

要将几何体的坐标从主体模型的系统转换为链接模型的系统，我们必须改用 **linkModelTransform** 的倒数。

Inverse Transform 对象包含“相反”的转换，并使用 linkModelTransform.inverse 属性获取。

### 链接模型的概念

在 Revit 中，链接模型是指将一个 Revit 项目文件插入到另一个项目中。这涉及到不同的坐标系：

- **主机模型坐标系**: 主项目文件的坐标系
- **链接模型坐标系**: 链接文件自身的坐标系
### Revit链接模型的类结构

链接模型由两个类表示：

1. **RevitLinkType**: 定义链接的类型信息，如文件路径、显示设置等
2. **RevitLinkInstance**: 表示链接实例在主机项目中的具体位置和变换

### 坐标转换方法

#### 从链接模型转换到宿主模型

```csharp
// 获取链接实例的变换矩阵
Transform linkToHostTransform = revitLinkInstance.GetTransform();
  
// 将链接模型中的点转换到主机坐标系
XYZ pointInHost = linkToHostTransform.OfPoint(pointInLink);

// 将链接模型中的向量转换到主机坐标系
XYZ vectorInHost = linkToHostTransform.OfVector(vectorInLink);
```

#### 从宿主模型转换到链接模型

```csharp
// 获取逆变换矩阵
Transform hostToLinkTransform = revitLinkInstance.GetTransform().Inverse;

// 将主机模型中的点转换到链接坐标系
XYZ pointInLink = hostToLinkTransform.OfPoint(pointInHost);

// 将主机模型中的向量转换到链接坐标系
XYZ vectorInLink = hostToLinkTransform.OfVector(vectorInHost);
```

### 实际应用示例

  
```csharp
public class CoordinateConversion
{
    // 将链接模型中的实体转换到主机模型
    public static Solid ConvertSolidFromLinkToHost(RevitLinkInstance linkInstance, Solid linkSolid)
    {
        Transform transform = linkInstance.GetTransform();
        return SolidUtils.CreateTransformed(linkSolid, transform);
    }

    // 将主机模型中的实体转换到链接模型
    public static Solid ConvertSolidFromHostToLink(RevitLinkInstance linkInstance, Solid hostSolid)
    {
        Transform transform = linkInstance.GetTransform().Inverse;
        return SolidUtils.CreateTransformed(hostSolid, transform);
    }

    // 批量转换点集
    public static List<XYZ> ConvertPointsFromLinkToHost(RevitLinkInstance linkInstance, List<XYZ> linkPoints)
    {
        Transform transform = linkInstance.GetTransform();
        return linkPoints.Select(p => transform.OfPoint(p)).ToList();
    }
}
```

### 变换矩阵的内部结构

Revit 的 Transform 矩阵是一个 3×4 仿射变换矩阵：

```
| BasisX.x  BasisY.x  BasisZ.x  Origin.x |   (X方向的分量)

| BasisX.y  BasisY.y  BasisZ.y  Origin.y |   (Y方向的分量)

| BasisX.z  BasisY.z  BasisZ.z  Origin.z |   (Z方向的分量)
```


对于链接模型变换：

- **Origin**: 链接模型原点在主机模型中的位置
- **BasisX, BasisY, BasisZ**: 链接模型坐标轴在主机模型中的方向
- **如果链接模型被旋转**，基向量会相应旋转
- **如果链接模型被镜像**，变换可能包含反射分量

## 高级应用技巧

### 1. 组合多个变换

```csharp
public static Transform CreateComplexTransform(XYZ translation, XYZ rotationAxis, double angle, XYZ rotationCenter)
{
    Transform t1 = Transform.CreateTranslation(translation);
    Transform t2 = Transform.CreateRotationAtPoint(rotationAxis, angle, rotationCenter);
    return t2.Multiply(t1); // 先平移，后旋转
}
```

### 2. 链式变换处理

```csharp
public static Solid ApplyMultipleTransforms(Solid solid, List<Transform> transforms)
{
    Solid result = solid;
    foreach (Transform transform in transforms)
    {
        result = SolidUtils.CreateTransformed(result, transform);
    }
    return result;
}
```

### 3. 变换的验证和检查

```csharp
public static void AnalyzeTransform(Transform transform)
{
    // 检查是否为刚体变换（无缩放，无反射）
    bool isRigidBody = Math.Abs(transform.Determinant - 1.0) < 1e-9 &&
                       !transform.HasReflection;
    // 提取平移分量
    XYZ translation = transform.Origin;
    
    // 提取旋转信息
    double angle = ExtractRotationAngle(transform);
    XYZ axis = ExtractRotationAxis(transform);
}
```

## 结论

除了将坐标从一个系统转换为另一个系统之外，将变换应用于几何形状还代表了更复杂任务的基础