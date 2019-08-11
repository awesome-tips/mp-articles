## Sorting Algorithms: Implementing Heap Sort Using Swift

| 作者：Jimmy M Andersson
| 链接：https://medium.com/appcoda-tutorials/sorting-algorithms-implementing-heap-sort-using-swift-f24e6868ad28

排序是计算机的一项主要任务。这并不是因为排序本身非常有趣，而是因为很多其它算法依赖于排序才能正常运行。本文主要描述如何实现堆排序算法，该算法依赖于称为堆的数据结构。

本文的具体实现可以查看对应的 [XCode Playground 文件](https://github.com/JimmyMAndersson/HeapSort)。

## 堆

堆是一个完整且部分排序的二叉树。通俗一点讲就是它总是将新数据节点插入最深一层的左侧。从某种意义上讲，元素间是有顺序的，尽管实际上并未做排序操作。

在本教程中，我们将使用最大堆（Max Heap）实现一个始终在 O(n * log(n)) 时间内运行的排序算法。首先，我们将实现一个 `SortingAlgorithms` 类，该类带有一个对数组进行排序的静态函数，然后再使用面向协议的解决方案。

关于最大堆，有三个重要事项：

* 始终保证树根中的元素值最大；
* 任何一个节点如果有子节点的话，它的值总是大于它所有的子节点的值；
* 堆通常可以以数组的形式实现，其中可以使用非常简单的数学公式来计算特定索引的父节点和子节点。这将使我们能够实现快速有效的排序。

## 代码

首先，我们来扩展 Swift 标准库的 Int 类型的实现，以抽象出获取父节点和子节点索引的运算公式。如下代码所示：

```swift
private extension Int {
  var parent: Int {
    return (self - 1) / 2
  }
  
  var leftChild: Int {
    return (self * 2) + 1
  }
  
  var rightChild: Int {
    return (self * 2) + 2
  }
}
```

使用这些代码，我们可以通过计算属性来计算索引，而不是需要时直接用数学公式来计算。这样保证了可读性。另外这是一个私有扩展，意味着它不会影响 Int 的整体性，而只是在该文件作用域中可用。

接下来，让我们分步来说明排序算法的各个步骤。

首先，我们将从数组构建一个最大堆。基本操作是将新元素插入到堆的末尾再交换到正确的位置，因此我们可以使用简单的循环来模拟插入操作。我们先假定数组只有两个元素，“堆积”这两个元素，在循环的每个迭代中，我们插入一个元素并重新堆积。如下所示：

![](https://cdn-images-1.medium.com/max/1600/1*0Pa1SrS8m7rWWVdIOilGhg.png)

操作完成后，数组看上去并没有特意排序。事实上，看起来更糟糕。这是因为我们使用数组存储了树的节点。看一下操作前后的对比：

![](https://cdn-images-1.medium.com/max/1600/1*lIRcKD_jWwWogWR-MIZ_jg.png)

看上去我们只是交换了元素，但事实上，我们刚刚建立了一个将用于完成排序的属性。

获取第一个元素，然后将其与最后一个元素交换，我们可以把最大的元素放到最后。然后假定数组长度减 1，然后重新堆积这个子数组，又可以得到这个子数组的最大元素。然后将子数组的最大元素放到子数组最后，这样依此类推，就可以得到一个完全排序的数组。

![](https://cdn-images-1.medium.com/max/1600/1*rjFm-5_kx3xFs0OQcNB3FA.png)

我们的 .heapSort(_:) 方法的代码如下，包括构建和缩小堆。

```swift
class SortingAlgorithms {
  private init() {}
  
  public static func heapSort<DataType: Comparable>(_ array: inout [DataType]) {
    if array.count < 2 { return }
    buildHeap(&array)
    shrinkHeap(&array)
  }
  
  private static func buildHeap<DataType: Comparable>(_ array: inout [DataType]) {
    for index in 1..<array.count {
      var child = index
      var parent = child.parent
      while child > 0 && array[child] > array[parent] {
        swap(child, with: parent, in: &array)
        child = parent
        parent = child.parent
      }
    }
  }
  
  private static func shrinkHeap<DataType: Comparable>(_ array: inout [DataType]) {
    for index in stride(from: array.count - 1, to: 0, by: -1) {
      swap(0, with: index, in: &array)
      var parent = 0
      var leftChild = parent.leftChild
      var rightChild = parent.rightChild
      while parent < index {
        var maxChild = -1
        if leftChild < index {
          maxChild = leftChild
        } else {
          break
        }
        if rightChild < index && array[rightChild] > array[maxChild] {
          maxChild = rightChild
        }
        guard array[maxChild] > array[parent] else { break }
        
        swap(parent, with: maxChild, in: &array)
        parent = maxChild
        leftChild = parent.leftChild
        rightChild = parent.rightChild
      }
    }
  }
  
  private static func swap<DataType: Comparable>(_ firstIndex: Int, with secondIndex: Int, in array: inout [DataType]) {
    let temp = array[firstIndex]
    array[firstIndex] = array[secondIndex]
    array[secondIndex] = temp
  }
}
```

这就是我们想要的，不过我们可以让它更干净一些。

## 面向协议的实现

通过扩展可比较元素类型的 Array 类型，我们能得到一些好处。一个是代码量更少，另一个是可以直接在对象上调用方法，而不需要如下处理：

```swift
SortingAlgorithms.heapSort(&myArray)
```

而是这样：

```swift
myArray.heapSort()
```

这样更加清晰。元素类型不符合 Comparable 协议时，编辑器甚至不会在数组对象上智能提示 .heapSort()。

```swift
public extension Array where Element: Comparable {
  public mutating func heapSort() {
    buildHeap()
    shrinkHeap()
  }
  
  private mutating func buildHeap() {
    for index in 1..<self.count {
      var child = index
      var parent = child.parent
      while child > 0 && self[child] > self[parent] {
        swapAt(child, parent)
        child = parent
        parent = child.parent
      }
    }
  }
  
  private mutating func shrinkHeap() {
    for index in stride(from: self.count - 1, to: 0, by: -1) {
      swapAt(0, index)
      var parent = 0
      var leftChild = parent.leftChild
      var rightChild = parent.rightChild
      while parent < index {
        var maxChild = -1
        if leftChild < index {
          maxChild = leftChild
        } else {
          break
        }
        if rightChild < index && self[rightChild] > self[maxChild] {
          maxChild = rightChild
        }
        guard self[maxChild] > self[parent] else { break }
        
        swapAt(parent, maxChild)
        parent = maxChild
        leftChild = parent.leftChild
        rightChild = parent.rightChild
      }
    }
  }
}
```

Done!!!

推荐阅读
如何阅读苹果开发文档
Dart vs Swift
Swift 5 新特性一览



