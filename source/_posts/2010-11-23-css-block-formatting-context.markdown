---
comments: true
date: 2010-11-23 00:35:16
layout: post
title: CSS的block formatting context
wordpress_id: 83
categories: [web_dev]
tags:  [bfc, css]

---

**什么是块格式化内容：Block Formatting Context （BFC）**
网页布局的时候，BFC会在常规流(与之并列的是浮动和绝对定位)中进行布局。BFC提供了一个环境，使得html元素在该环境内按照一定规则进行布局，而不会影响其它环境中的布局

**什么情况下生成BFC**

当一个元素满足以下条件时，它的包含块将会形成一个BFC：



	
  * 浮动(float属性非none)

	
  * overflow属性的计算值不是visible

	
  * display属性是table-cell、table-caption或着inline-block

	
  * 绝对定位(position非static且非relative)


需要注意的是，disaply属性为table时并不会建立一个BFC。但是由于display为table时，元素将会生成一个匿名框，这些框会具有display:table-cell属性，故会建立新的BFC。也就是说，建立BFC的是这个display为table-cell的匿名框而非table。所以在display:table和display:table-cell的元素上使用clear属性的效果是不一样的（见最后一小节示例）。
**BFC的影响**



	
  1. rule 1. 在bfc内部，垂直相邻的框的margin将会折叠(collapse)

	
  2. rule 2. 在同一个bfc内部，所有框的左外边均挨着包含块(bfc)的左侧(rtl时,是右外边挨着包含块的右侧)。即使该bfc中包含float元素也是如此，除非该bfc内部另外一个元素形成了一个新的bfc，这种情况下，该元素的框将会挨着float元素排列，宽度可能会由于float元素的存在而变窄

	
  3. rule 3. 一个BFC的border-box不能和与其在同一个BFC的float元素的margin-box相互重叠。这就意味着，浏览器会在BFC上生成不可见的margin来避免它和float元素的margin-box相互重叠。所以，在紧接着float元素的BFC上设置负的margin是没有作用的。而且BFC的margin还是可以和float元素重叠的，不能重叠的只是border-box。

	
  4. rule 4.  clear只能清除处于同一个bfc的元素


**Rule 1 BFC and margin collapse**
从rule 1可知，垂直相邻的框的margin发生collapse的情况只会发生在这两个相邻框在同一个bfc的情形下，而如果两个垂直相邻的框分属于不同的bfc,则不会发生折叠。
如下例:

[html]
<h3>margin collapse in the same BFC:</h3>

 <div style="background-color:skyBlue">
     <p style="margin-bottom:20px;border:1px solid red">
        this is text 1 with style: <br/> {margin-bottom:20px;border:1px solid red}
     </p>
 </div>
 <div style="background-color:skyBlue">
     <p style="margin-top:10px;border:1px solid black">
        this is text 2 with style: <br/> {margin-top:10px;border:1px solid black"}
     </p>
 </div>

 <h3>margin not collapse in different BFCs</h3>
 <div style="background-color:skyBlue">
     <p style="margin-bottom:20px;border:1px solid red">
        this is text 3 with style: <br/> {margin-bottom:20px;border:1px solid red}
     </p>
 </div>
 <div style="background-color:skyBlue;overflow:hidden">
     <p style="margin-top:10px;border:1px solid black">
        this is text 4 with style: <br/> {margin-top:10px;border:1px solid black"} <br/>
        <b>but the containing div has property: "overflow:hidden", make this div establish a new BFC</b>
     </p>
 </div>
[/html]

效果如下图所示，前两个段落中间的margin发生了collapse，变为了较大的20px，而下面两个段落并未发生collapse，总共是30px
![](http://leivli.duapp.com/wp-content/uploads/2010/11/12-1024x286.png)

因此在需要避免margin collapse时，可以通过构建BFC来实现，这也是为什么浮动的元素和绝对定位的元素不会发生margin collapse的原因了

**Rule 2 float in the BFC**

如果一个无top/bottom border、无top/bottom padding且不形成bfc的div内部包含多个float元素，由于float元素是脱离文档流的，所以即使该div具有margin，也会由于margin相邻而会发生collapse，从而形成一个零高度的div。如果多个这样的div相邻，如果水平空间允许，那么这些div中的float元素将会排列在同一行上，这可能并非预期的效果。如本意要布局一个两行两列div的如下代码片段:

[html]
 <h3>float in zero height divs of the same BFC</h3>

 <div style="margin:20px">
      <div style="float:left;border:1px solid green;background-color:red">float div r1c1</div>
      <div style="float:left;border:1px solid green;background-color:blue">float div r1c2</div>
 </div>

 <div style="margin:20px">
      <div style="float:left;border:1px solid green;background-color:gray">float div r2c1</div>
      <div style="float:left;border:1px solid green;background-color:yellow">float div r2c2</div>
 </div>
[/html]

将生成下列效果的页面,四个浮动框会排列在一行上，而非预期的两行两列布局:

![](http://leivli.duapp.com/wp-content/uploads/2010/11/2.png)

由rule 2可知，在同一个BFC内部，所有框的左外边会挨着左侧排列，即使包含float元素也是如此。因此，如果将上例中的两个div各自均改为BFC，就会达到预期的目的。由于bfc内部两个float元素均会各自生成一个bfc，这两个bfc就会并排排列（如果bfc宽度足够的话，如果不够，由于都是float，第二个float会下移)。而这两个不同BFC又处于初始包含块的BFC中，将会按照常规流进行布局，即第二个BFC排在第一个BFC下面，形成2行2列的样式：

    
    <span style="text-decoration: line-through;">
    <h3>float in zero height divs of the same BFC</h3>
    
     <div style="margin:20px">
    <div style="float:left;border:1px solid green;background-color:red">float div r1c1</div>
    <div style="float:left;border:1px solid green;background-color:blue">float div r1c2</div>
     </div>
    
     <div style="margin:20px">
    <div style="float:left;border:1px solid green;background-color:gray">float div r2c1</div>
    <div style="float:left;border:1px solid green;background-color:yellow">float div r2c2</div>
     </div></span>
    [html]
    <h3>float in zero height divs of different BFCs</h3>
     <div style="margin:20px;overflow:hidden">
          <div style="float:left;border:1px solid green;background-color:red">float div r1c1</div>
          <div style="float:left;border:1px solid green;background-color:blue">float div r1c2</div>
     </div>
     <div style="margin:20px;overflow:hidden">
          <div style="float:left;border:1px solid green;background-color:gray">float div r2c1</div>
          <div style="float:left;border:1px solid green;background-color:yellow">float div r2c2</div>
     </div>
    
    [/html]
    
    
    
    实际布局效果如下图：
    
    <img src="http://leivli.duapp.com/wp-content/uploads/2010/11/3.png" title="float_in_bfc" height="131" width="519" alt="" class="aligncenter size-full wp-image-92"></img>
    
    <strong>Rule 3 BFC  adjoin float</strong>
    
    一个普通的div元素(未生成新的BFC)是会被相邻的float元素覆盖的，而一个生成BFC的div元素的border-box是不会被相邻的float元素覆盖的，但是其margin可以被覆盖。如下面的代码：
     [html]
    
    <h3>float and normal div overlap</h3>
     <div style="float:left;background-color:gray">float div</div>
     <div style="background-color:yellow;border:1px solid red"> normal div </div>
    
     <h3>float and bfc div no overlap</h3>
     <p>both divs have no margin</p>
     <div style="float:left;background-color:gray">float div</div>
     <div style="background-color:yellow;border:1px solid red;overflow:hidden"> bfc div </div>
    
     <p>bfc div has margin</p>
     <div style="float:left;background-color:gray">float div</div>
     <div style="background-color:yellow;border:1px solid red;overflow:hidden;margin-left:10px"> bfc div </div>
    
     <p>float div has margin</p>
     <div style="float:left;background-color:gray;margin-right:10px;margin-right:10px">float div</div>
     <div style="background-color:yellow;border:1px solid red;overflow:hidden"> bfc div </div>
    
    [/html]
    
    
    
    生成的效果如下图：
    
    <img src="http://leivli.duapp.com/wp-content/uploads/2010/11/4-1024x209.png" title="float_adjoin_bfc" height="209" width="1024" alt="" class="aligncenter size-large wp-image-93"></img>如图所示，前两个div中，float的div会和另一个div发生重叠，而后面三个示例可以看出，float框不会和相邻的bfc的border box发生重叠。正如rule 3所述，float和bfc的border box不会发生重叠：
    当float和bfc都没有margin时，两者紧邻；当bfc有left margin时，该left margin可以被float框覆盖，两者还是紧邻；当float框有right margin时，bfc并不能将其覆盖。
    
    再看另外一个例子，不多说，贴代码:
     [html]
    
    <h3>float and bfc div no overlap: another example</h3>
    
     <div style="overflow:hidden">
    <div style="float:left;background-color:red;width:200px">first: left float div</div>
    <div style="float:right;background-color:green;width:200px">second: right float div</div>
    <div style="overflow:hidden;background-color:yellow">third: bfc div</div>
     </div>
    
     <div style="overflow:hidden">
    <div style="float:left;background-color:red;margin-right:20px;width:200px">first: left float div</div>
    <div style="float:right;background-color:green;margin-left:20px;width:200px">second: right float div</div>
    <div style="overflow:hidden;background-color:yellow">third: bfc div</div>
     </div>
    
     <div style="overflow:hidden">
    <div style="float:left;background-color:red;width:200px">first: left float div</div>
    <div style="float:right;background-color:green;width:200px">second: right float div</div>
    <div style="overflow:hidden;background-color:yellow;margin:0 220px">third: bfc div</div>
     </div>
    
    [/html]
    
    
    
    实际运行效果图：
    
    <img src="http://leivli.duapp.com/wp-content/uploads/2010/11/5-1024x95.png" title="float_adjoin_bfc_another" height="95" width="1024" alt="" class="aligncenter size-large wp-image-94"></img>
    
    该例子中有三组，各组均有三个div，分别为float left（width:200px）、float right（width:200px）和无float的third div（一个bfc），每组的包含div具有overflow:hidden，故会形成一个bfc，
    即各组的三个div均处于一个独立的BFC中。
    
    第一组中，根据Rule3， third div形成的bfc不会和float div发生重叠，因此在水平空间足够的情况下，三个div会排在一行，中间没有间隙。此时在third div上应用margin并不会在third div的两边和float div
    交界处形成空隙。因为这种情况下，如果需要在中间框的左右两侧产生空隙，通过给该框添加margin是无法做到的，但是可以采用以下两个方法做到：
    
    <ul>
    
    	<li> 在两边的float上设置margin，如上图第二组所示</li>
    
    
    	<li> 给third div设置一个大于两边float框宽度的margin，如上图第三组所示</li>
    
    </ul>
    
    
    <strong>Rule 4 BFC and clear</strong>
     clear只能清除处于同一个bfc的float元素，这里给出的例子说明了该点，同时也表示了第一段提到的在table元素和table-cell元素上使用clear的不同:
     [html]
    
    <h3>BFC & clear -- clear on display:talbe vs clear on display:table-cell</h3>
    
     <p style="clear:left">normal div after float div without clear</p>
     <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="border:2px solid yellow">div content</div>
     </div>
    
     <p style="clear:left">normal div after float div with clear</p>
     <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="border:2px solid yellow;clear:left">div content</div>
     </div>
    
     <p style="clear:left">display:table after float div without clear</p>
     <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="display:table;border:2px solid yellow"><span>div content</span></div>
     </div>
    
     <p style="clear:left">display:table after float div with clear</p>
    <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="display:table;clear:left;border:2px solid yellow">div content</div>
     </div>
    
     <p style="clear:left">display:table-cell after float div without clear</p>
     <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="display:table-cell;border:2px solid yellow">div content</div>
     </div>
    
    <p style="clear:left">display:table-cell after float div with clear</p>
     <div style="border:1px solid red">
    <p style="float:left;background-color:gray;color:white;border:1px solid green">
    line 1<br/>
    line 2
    </p>
    <div style="display:table-cell;clear:left;border:2px solid yellow">div content</div>
    </div>
    [/html]
    
    
    
    实际运行效果图：
     <img src="http://leivli.duapp.com/wp-content/uploads/2010/11/6-1024x362.png" title="bfc_and_clear" height="362" width="1024" alt="" class="aligncenter size-large wp-image-96"></img>第一组是一个float div后跟着一个普通的无设置clear属性的div（无display属性）的情况，此时float div会覆盖后面的div，从边框可以看到，后面的div还是从包含div的左侧开始的
     第二组和第一组一样，只是后面的div设置了clear:left属性，这就使得后面的div排列在了float div的下面
     第三组中后面的div设置了display:table,但是没有设置clear属性，与第一组相比，该div的表现更像是一个ＢＦＣ，因为该div没有从包含div的左侧开始，而是从float div的右侧开始布局。
     这就是文章刚开始提到的，display:table本身不会新建一个bfc，但是由于它会使得内部元素生成display:talbe-cell的匿名框，而该匿名框会建立一个bfc
     第四组可以看出第三组中的bfc并不是display:table创建的，因为设置了clear:left后，该div到了float div的下面。
     最后两组是display:table-cell的情况，此时后面的div会建立一个bfc，由于和float的div分属不同的div，clear属性是不起作用的，这也就是为什么第五组设置了clear属性后，表现和第四组一样的原因
    
