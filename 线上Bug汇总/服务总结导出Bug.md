# 服务总结导出Bug

> 服务总结导出的bug

### 场景：

服务总结导出，涉及到单元格的合并。   在处理完每一行的数据后，需要对同一个session的单元格进行合并，合并后出现数据丢失。



### 方案

经过debug排查，发现是如下代码出现问题

```java
if(treeValueListSize > 1){
                for(int n = 0; n < firstTitleNodeIndex;  n++){
                    CellRangeAddress region = new CellRangeAddress(dataRow, dataRow + treeValueListSize-1, n, n);
                    sheet.addMergedRegion(region);
                }
                for(int n = lastTitleNodeIndex + 1; n < title.length;  n++){
                    CellRangeAddress region = new CellRangeAddress(dataRow, dataRow + treeValueListSize-1, n, n);
                    sheet.addMergedRegion(region);
                }

                dataRow += treeValueListSize;
            }
```

这里涉及到 `sheet.addMergedRegion(region)` 这个方法，方法如下

```java
sheet.addMergedRegion(new CellRangeAddress(int firstRow, int lastRow, int firstCol, int lastCol)
```

代码从这里走过后， sheet里面的rows出现混乱。 初步猜测是这里的问题。

**那怎么办呢？**

猜想： 是在处理每一组数据的同时，进行excel行合并，导致出现问题。

**方案**

1. 重写合并方法，将所有excel行都生成后，进行标记合并。  对每一组数据，记录要合并的 行号和列号。 修改。       **成功！**

2. 仔细研究之前的代码，网上搜，边处理数据，边合并单元格。  也搜不到。   合并的单元格 为 region,  却在debug的里面找不到。。。。   放弃