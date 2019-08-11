| 作者：Jimmy M Andersson
| 链接：https://medium.com/swlh/sorting-algorithms-implementing-merge-sort-using-swift-a1236a0be2b4

之前介绍了堆排序，这是一种基于堆的排序算法。今天，我们进一步深入研究排序算法，来看看归并排序，这是一种时间复杂度为 `O(n * log(n))` 的排序算法，作为权衡，其空间复杂度为 `O(n)`。

本文的具体实现可以查看相应的 [XCode Playground 文件](https://github.com/JimmyMAndersson/MergeSort)

## 什么是归并排序?

归并排序属性分治法一类的算法。分治法基本思想是解决只包含一半数据的问题要容易得多，因此可以尝试将数据递归地分割成两个大小相等的数据集来处理。

在归并排序中，对原始数组大小一半的数据进行排序要容易得多。将原始数组分割成四个四分之一大小的数组分别排序会更简单，因此我们执行递归分割操作。这是该算法巧妙的地方。由于我们处理的是有限大小的数组，所以最终分割得到的所有数组大小都会少于两个元素，而这样的数组可以看作是已排序的。

当到达上面所说的状态时，递归终止，我们开始展开调用堆栈。在这个过程中，我们通过比较每个数组中的元素，并将它们插入到一个新数组中来合并数组，然后将这些数组返回给调用堆栈的下一帧。来看看下面的草图，了解它的工作原理：

![](https://cdn-images-1.medium.com/max/1600/1*6OfWzD7sopTy2SkGZM6gdw.png)

## 构造算法

我们将采用面向协议的方法，这意味着我们将扩展包含符合 `Comparable` 协议的元素的数组的 `Array` 实现。我们还将实现一个返回数组的排序副本的方法，而不是替换我们调用方法的数组。如下代码所示：

```objc
extension Array where Element: Comparable {
 public mutating func mergeSort() {
    let startSlice = self[0..<self.count]
    let slice = mergeSort(startSlice)
    let array = Array(slice)
    self = array
  }
  
  public func mergeSorted() -> Array<Element> {
    let startSlice = self[0..<self.count]
    let slice = mergeSort(startSlice)
    let array = Array(slice)
    return array
  }
}
```

请注意，我们声明了一个名为 `startSlice` 的变量。我们将使用一个名为 `ArraySlice` 的泛型结构，它将帮助我们避免在调用堆栈中对数组进行不必要的复制。相反，我们将创建一个已分配内存的视图，并只在我们将两个切片合并在一起时才分配新内存。当对 `.mergeSort(_:)` 的调用返回时，我们需要将它转换为一个 `Array`，以便与我们的返回类型兼容。但是，请注意，Swift 编译器是一个非常智能的构造，它只是让新的 Array 对象使用 `ArraySlice` 中已经分配的内存，因此不需要任何开销。

接下来，我们定义了 `.mergeSort(_:)` 方法，它仍然在扩展范围内：

```objc
private func mergeSort(_ array: ArraySlice<Element>) -> ArraySlice<Element> {
    if array.count < 2 {
      return array
    } else {
      let midIndex = (array.endIndex + array.startIndex) / 2
      let slice1 = mergeSort(array[array.startIndex..<midIndex])
      let slice2 = mergeSort(array[midIndex..<array.endIndex])
      return merge(slice1, slice2)
    }
  }
```

这里包含我们决定是否可以将数组的分割部分视为“已排序”的部分。如果数组的元素少于两个元素，则按定义是已排序的，因此我们只返回相同的数组。如果它包含更多元素，我们通过创建两个新的 `ArraySlice` 对象并将它们传递给递归调用再次拆分它。当两个递归调用返回时，我们可以确定我们有两个排序的数组切片可以使用，所以我们调用方法 `merge(_:_:)` 将两个切片合并为一个并返回它。 `merge(_:_:)` 看起来像这样：

```objc
private func merge(_ firstArray: ArraySlice<Element>, _ secondArray: ArraySlice<Element>) -> ArraySlice<Element> {
    var newArray = ArraySlice<Element>()
    newArray.reserveCapacity(firstArray.count + secondArray.count)
    var index1 = firstArray.startIndex
    var index2 = secondArray.startIndex
    
    while index1 < firstArray.endIndex && index2 < secondArray.endIndex {
      if firstArray[index1] < secondArray[index2] {
        newArray.append(firstArray[index1])
        index1 += 1
      } else {
        newArray.append(secondArray[index2])
        index2 += 1
      }
    }
    
    if index1 < firstArray.endIndex {
      let range = index1..<firstArray.endIndex
      let remainingElements = firstArray[range]
      newArray.append(contentsOf: remainingElements)
    }
    if index2 < secondArray.endIndex {
      let range = index2..<secondArray.endIndex
      let remainingElements = secondArray[range]
      newArray.append(contentsOf: remainingElements)
    }
    
    return newArray
  }
```

这个有点棘手，所以请花时间阅读并理解它。我们首先创建一个新的 `ArraySlice` 对象和两个变量，以跟踪我们当前在每个切片的索引。然后我们比较每个切片的元素，将最小的元素放入我们的新 `ArraySlice` 对象中，直到完成其中一个原始切片。最后，我们检查是否还有剩余的元素需要插入。一旦完成，我们将新切片返回到调用函数，我们就完成了排序操作。

