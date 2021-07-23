topic是逻辑上的概念，partition是物理上的概念

目录：topic-分区号

内部存储是index+log 逻辑上有segment的概念，查找数据是二分查找segment，因为segment的命名规则是上一个segment的最后一个offset+1





