---
title: 记一次Excel导入内存溢出的优化
date: 2021-01-05 14:27:29
categories: Java
tags: 内存 Excel 导入
---

### 背景

​		`Excel`的导入导出功能在一个独立的应用中，各系统都在用，而且分配的内存有限，`JVM`给了 1G 多点，所以动不动就内存溢出了。至于为什么不调整内存，可能是强制要求写低内存的代码，上面大佬们的决定，也不好说什么，只能默默承受。

​		这不，业务刚导入了一个一个`3M`的`Excel`文件，系统监控立马告警`OOM`异常。为了保证下一次业务导入不出错，这活儿落到了我手上。

​		首先看了一下`dump`文件，溢出的原因的数组的扩容。也就是集合在不断扩容的过程中，导致`Eden Space`放不下新对象了，然后向`Old Gen`求救，结果`Old Gen`也放不下了，就抛异常了，唉，可怜的虚拟机，承担了太多。

​		集合不断扩容是因为`Excel`的解析，虽然`3M`的文件听起来不大，但是将里面的每个工作表，每行和每列转换成对象以后就不止是`3M`了。使用的工具库为`POI`，这个大家应该也熟悉，将`Excel`文件转换成`XML`字节流，然后使用`DOM`解析的方式将整个字节流转换成`Java`对象，全部放到内存中。

​		总结一下，内存溢出的原因主要是下面几点：

		1. JVM 的内存本来就小
		2. 大数据量文件解析后被全部放到了 JVM Heap 中

### 方案

1. 调整`JVM`内存，这个行不通的，按上面大佬的意思，给多大内存，如果代码不注意，都会溢出。所以这个方案是行不通的。

2. 修改解析方式，`DOM`解析比较简单暴力，会将整个`XML`读入内存并构建一个`DOM`树，基于这棵树形结构对各个节点进行操作。`XML`文档中的每个成分都是一个节点：整个文档是一个文档节点，每个`XML`标签对应一个元素节点，包含在 `XML`标签的文件是文本节点，每一个`XML`属性是一个属性节点。 基于`DOM`树可以向上或者向下检索元素，缺点就是比较占内存。

	`POI`还提供了基于事件模型的解析方式`SAX`，它并不需要将整个`XML`文档加载到内存中，而只需将`XML`文档的一部分加载到内存中，即可开始解析，在处理过程中并不会在内存中记录`XML`中的数据，所以占用的资源比较小。当程序处理过程中满足条件时，也可以立即停止解析过程，这样就不必解析剩余的`XML`内容。当解析到某类型节点时，会触发注册在该节点上的回调函数，我们可以根据自己的业务需求注册相应事件的回调函数，缺点就是要自己维护节点间的关系。

3. 使用`Redis`进行风险转移，将解析后的数据和校验错误信息放入`Redis`中，保证`JVM`不会存在过多的对象。

### 执行

​		修改解析方式，使用`SAX`解析方式解析`Excel`文件，将解析后的数据转换成对应的`Dto`放入集合中，这里要注意的是，如果数据量大的话，集合也会扩容，有溢出的危险，所以这里限定集合的数量到达`1000`后放入`Redis`中。

```java
		InputStream sheetStream = null;
		OperationContentHandler operationContentHandler = null;
		List<DataDto> dataList = new ArrayList<>();
		try (OPCPackage opk = OPCPackage.open(in)){
			//SAX解析excel
			XSSFReader reader = new XSSFReader(opk);
             //共享样式表
			StylesTable stylesTable = reader.getStylesTable();
             //只读共享字符串表
			ReadOnlySharedStringsTable sharedStringsTable = new ReadOnlySharedStringsTable(opk);
			//获取到要解析的sheet流，XSSFReader在解析时会将sheet页按rId + sheetIndex排序，从1开始，当前要解析的数据在第二个sheet页，所以这里是rId2
             sheetStream = reader.getSheet("rId2");
			//创建解析器
			XMLReader parser = SAXHelper.newXMLReader();
			//创建内容解析处理器，这个处理器是我自己实现的
			operationContentHandler = new OperationContentHandler(cacheService, dataList, cacheKey);
             //绑定处理器
             parser.setContentHandler(new XSSFSheetXMLHandler(stylesTable, sharedStringsTable, operationContentHandler, false));
             //解析sheet流
			parser.parse(new InputSource(sheetStream));
		} catch (IOException | SAXException | OpenXML4JException | ParserConfigurationException e) {
			throw new Exception("导入数据解析出错：", e);
		} finally {
			if (sheetStream != null) {
				try {
					sheetStream.close();
				} catch (IOException e) {
					logger.error("解析出错,流关闭失败", e);
				}
			}
		}
```

​		自己实现的内容处理器，实现`XSSFSheetXMLHandler.SheetContentsHandler`接口 ，主要是在解析`row`开始时，解析`row`结束时，解析每一个`cell`时添加自定义处理。其中大部分解析的处理都被`XSSFSheetXMLHandler`承包了，核心逻辑就是解析`XML`节点，有兴趣的看下源码。

```java
class ContentHandler implements XSSFSheetXMLHandler.SheetContentsHandler{
    
    private List<DataDto> dataList;
    private DataDto dataDto;
    /**
    * 解析行之前，参数是当前行的行号（从0开始）
    **/
    public void startRow(int rowNum){
        dataDto.setRowNum(rowNum);
    };
    /**
    * 解析行之后，参数是当前行的行号（从0开始）
    **/
    public void endRow(int rowNum){
        //这里所做的处理是解析行之后会判断解析的数据是否到达了一定的数量，比如10000
        //然后将DataDto转换为字符串放入了redis
        if (CollUtil.isNotEmpty(dataList) && dataList.size() == 1000){
				List<DataDto> notEmptyList = dataList.parallelStream()
                    						.filter(dto -> !dto.isEmpty())
                    						.collect(Collectors.toList());
				if (notEmptyList.size() != dataList.size()){
					dataList = notEmptyList;
					return;
				}
				count += dataList.size();
				cacheService.rpush(cacheKey, JSON.toJSONString(notEmptyList));
				cacheService.expire(cacheKey, REDIS_EXPIRE_TIME);
				dataList.clear();
			}
    }
    /**
    * 解析单元格
    * cellReference 单元格的坐标，参考excel表格中坐标，类似于A1,B2,C33这种，第一行从1开始，第一列是A
    * formattedValue 单元格内容
    * comment 单元格注释
    **/ 
    public void cell(String cellReference, String formattedValue, XSSFComment comment){
        
    }
    /**
    * 解析页眉或页脚
    **/
     public void headerFooter(String text, boolean isHeader, String tagName){
         //没做处理
     }
}
```

​		这里要注意的是，存入`redis`中的`key`一般是要设置一个失效时间的，不能依赖于其自身内存淘汰策略，尤其是这种大批量数据的情况，一是要设置失效时间，二是在必要操作完成之后执行删除操作。

​		在我写的`DataDto`里面，是有设置对应的行号，这样方便定位哪一行数据出现了问题。

### 测试

​		之前使用`DOM`解析时，使用的测试参数为1024M，使用业务提供的`Excel`文件导入之后，马上就报`OOM`了。后续改为了`SAX`解析，使用400M的内存也毫无压力，甚至还有剩余，算是初步解决了这个导入的问题。

```
-Xms1024M
-Xmx1024M
-XX:+PrintGCDetails
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=D:/dump
-Xloggc:D:/dump/heap_log.txt
```

### 小结

1. 开发时一定要注意大数量的情况，集合无节制的扩容是很容易导致内存溢出，同时扩容的动作也会损耗性能，初始化集合时尽量设定一个初始值。
2. `DOM`和`SAX`解析都有其优越性，但也存在缺点，要分情况选择合适的解析方式。
3. `Redis`的存储要避免非热点数据长时间占据内存，设置超时时间和及时删除`key`，每一块内存都很珍贵，要好好利用。

