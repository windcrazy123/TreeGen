# TreeGen
**Unrean Engine Procedural Mesh Component Generate Trees**

函数所在位置：`Engine\Plugins\Runtime\ProceduralMeshComponent\Source\ProceduralMeshComponent\Private\KismetProceduralMeshLibrary.cpp`
`CreateGridMeshTriangles`是简单的一种构建vertex index的方法，这和x，y的循环嵌套顺序有关
首先他使用两个for循环嵌套分出一个个四边形，再把这个四边形的index按照顺序添加到Triangles
```cpp
Triangles.Add(Vert0);
Triangles.Add(Vert1);
Triangles.Add(Vert3);

Triangles.Add(Vert1);
Triangles.Add(Vert2);
Triangles.Add(Vert3);
```

`CalculateTangentsforMesh`函数, 此函数执行消耗可以被命令行`stat ProceduralMesh`捕获
首先声明一些变量做准备,比如三角形面数、顶点数、每个三角面的TBN还有
```cpp
// key是Vertices数组的index即VertexIndex，value是三角形面的index
TMultiMap<int32, int32> VertToTriMap;
// 和上面类似，不过会对重叠的点做一些处理来使后面计算的normal更平滑
TMultiMap<int32, int32> VertToTriSmoothMap;
```

之后使用传过来的Triangles数组计算得到三角面数，遍历每个三角面来填充这些变量
在每个三角面中首先遍历三角形的三个点，计算三角形顶点的index，然后处理重叠的点
```cpp
// 处理重叠的点
for (int32 OverlapIdx = 0; OverlapIdx < VertOverlaps.Num(); OverlapIdx++)
{
    // For each vert we overlap..
    int32 OverlapVertIdx = VertOverlaps[OverlapIdx];

    // 将同位置的点都添加当前三角形index
    VertToTriSmoothMap.AddUnique(OverlapVertIdx, TriIdx);

    // 将当前顶点index添加同位置点所在的三角形index
    TArray<int32> OverlapTris;
    VertToTriMap.MultiFind(OverlapVertIdx, OverlapTris);
    for (int32 OverlapTriIdx = 0; OverlapTriIdx < OverlapTris.Num(); OverlapTriIdx++)
    {
        VertToTriSmoothMap.AddUnique(VertIndex, OverlapTris[OverlapTriIdx]);
    }
}
```
然后三角形两边叉乘作为此三角形的法线，在通过纹理坐标矩阵的逆矩阵乘以三角形两边组成的矩阵得到矩阵(T,B)，乘以(1,0,0)得到T，乘以(0,1,0)得到B
>参考：https://learnopengl-cn.github.io/05%20Advanced%20Lighting/04%20Normal%20Mapping/

计算完所有三角形面后，再根据之前记录的`VertToTriMap`和`VertToTriSmoothMap`将同一个顶点上的TBN数据进行简单相加(无权，不是有权相加，比如可使用面积或夹角等方法进行加权，确保越大的三角形对顶点的影响越大)

最后归一化并使用施密特正交化确保T与N正交