## jQgrid 行内编辑
jQgrid是一个基于jQuery实现的表格插件，提供了十分丰富的API以供开发者使用，不过也由于版本更迭和本身支持的原因，在网络上查找一些相关资料的时候，总是会有许多种不同的解决方案。下面描述一个使用jQgrid3.8.2来实现的行内编辑功能。  
下面为进行行内编辑的Grid代码，可用的编辑单元格为’useCount’和’loadRate’. 单元格光标移除之后，将数据保存到本地Table。  

````javascript
//标识符
var lastsel = "";
var lastCol = "";
var countBlurMarkup = 0;
var rateBlurMarkup = 0;

var grid = $("#grid").jqGrid({
	//本地存储，手动对Grid进行reload操作
   	url:'',
	datatype: "local",
	height: 100,
	autowidth: true,
	scroll: 1,
	colNames:['id','model','weight','availableCount','useCount','loadRate'],
   	colModel:[
	    {name:'id',index:'id',sorttype:'string',editable:false,width:10,hidden:true},
   		{name:'model',index:'model',sorttype:'string',editable:false,width:50},
   		{name:'weight',index:'weight',sorttype:'number',editable:false,width:50},
   		{name:'availableCount',index:'availableCount',sorttype:'number',editable:false,width:50},
   		{name:'useCount',index:'useCount',sorttype:'number',editable:true,width:50,editoptions: {
             dataEvents: [
               { type: 'blur',
                   fn: function(e) {
                	   if(countBlurMarkup == 0) {
                		   $("#grid").jqGrid('saveRow',lastsel,lastCol);
                		   countBlurMarkup = 1;
                	   }
                	  
                   }
               }
            ]
        }}, {name:'loadRate',index:'loadRate',sorttype:'number',editable:true,width:50,editoptions: {
             dataEvents: [
               { type: 'blur',
                   fn: function(e) {
                	   if(rateBlurMarkup == 0) {
                		   $("#grid").jqGrid('saveRow',lastsel,lastCol);
                		   rateBlurMarkup = 1;
                	   }
                   }
               }
            ]
		}}
   	],
   	rowNum:100,
	loadonce:true,
   	mtype: "POST",
	multiselect: true,
	viewrecords: true,
	sortorder: "asc",
	caption: "CAP",
   	jsonReader: {root:"result",
        repeatitems : false
    },
    //双击操作
    ondblClickRow: function(rowId, rowIndex, columnIndex) {
    	if(rowId){
    		var grid = $('#grid');
    		grid.jqGrid('restoreRow',lastsel);
    		//获取编辑行的列焦点
    		grid.editRow(rowId, true, function() {
    		    var colModel = grid.getGridParam('colModel');
    		    var colName = colModel[columnIndex].name;
    		    var input = $('#' + rowId + '_' + colName);
    		    input.get(0).focus();
    		});
    		lastsel = rowId;
    		lastCol = columnIndex;
    		countBlurMarkup = 0;
    		rateBlurMarkup = 0;
    	}
    },
    //保存地址问本地Table
    editurl:'clientArray'
});
````

在完成这个例子的过程中，踩过几个坑

- 开始设置可编辑行，查阅jQgrid的API有发现一个`cellEdit:true`属性，在设置之后，的确是可以进行编辑操作。但是在编辑之后，若单元格还是处于编辑状态，则通过`getRowData`获取到的值为Input框，无法正确获取里面的值。解决的方法一为解析Input对象里面的`text`文本，将值重新填入获取到的`GridRowData`对象;一为先进行一次`saveRow`之后再进行获取;在实际的应用中，遇到多个可以编辑单元的时候解析对应的Input较为麻烦，第二个方法需要进行方法编辑状态结束判断，若无法得知则难以确定保存的时机。
- 基于上面的考虑，我决定还是使用`ondblClickRow`函数在双击Grid的时候，进入编辑模式，这里又涉及编辑焦点的定位问题，jQgrid的内部单元格id使用的是`$('#' + rowId + '_' + colName)` 模式，因此可以通过这个方法获取到双击的Input单元格，从而设置焦点。
- 编辑状态离开的判断主要使用jQgrid单元格编辑的`editoptions`和`dataEvents`属性，前者为编辑选项，通过查看API可以获取俩面的各种属性和对应的意义，dataEvents 为单元格的事件处理，可选的有 `click` `blur` `change`等，对应不同的单元格事件，这边使用`blur`监听鼠标焦点移除之后的事件，执行的方法为保存行。
- 在处理光标焦点移除保存的时候，发现执行`$("#grid").jqGrid('saveRow',lastsel,lastCol)`之后，如果保存当前行的其他列，会再次出发`blur`事件，(具体的原因还没有找到)。因此设置了两`markup` 标识，来进行判断，只有在开始编辑之后才会执行`$("#grid").jqGrid('saveRow',lastsel,lastCol)`的方法。避免了重复保存，一直无法进入编辑状态的情况。

因为使用jQgrid的时间很短，项目驱动本身没有太多的精力去深入研究jQgrid的原理和其实现， 主要以查看文档资料，头疼医头脚疼医脚的方法进行，对这个插件还是处于一知半解的情况， 可能有更好的解决问题的方法。但还是记录下来，以后备查。