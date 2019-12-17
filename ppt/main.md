title: 短信编辑器踩坑实践
speaker: pengyumeng
theme: dark
transition: move

[slide]

# 短信编辑器踩坑实践
pengyumeng

[slide]

# <font color=#0099ff>目录</font>

- 需求梳理

- 功能拆解

[slide]

# <font color=#0099ff>需求梳理</font>

![requirement](../img/requirement.jpg)

- 短信内容有签名前缀，签名颜色和正文有区别，且固定在前缀位置上，不可删除
- 可以插入短链和用户昵称，插入后可以整体删除
- 可以实现粘贴，但是只能粘贴文字
- 有退订标识（该内容实现较为简单，本次不讲解该内容）

[slide]

# <font color=#0099ff>功能拆解</font>

- 签名前缀的实现

- 插入用户昵称的实现

- 粘贴功能的重写

- 其他踩坑的地方

[slide]

# <font color=#0099ff>编辑器实现第一步</font>

给容器添加 contenteditable="true" 属性

[slide]

# <font color=#0099ff>签名前缀的实现</font>

[slide]

# <font color=#ff9933>踩坑1：通过浮动元素实现前缀</font>

通过 float: left 属性实现，使文字环绕，前缀位置也能固定

想法很美好，现实很骨感 - 初始化光标位置不正确

[slide]

# <font color=#0099ff>如何实现前缀的效果？</font>

- 固定给容器最前面添加一个 dom 元素（比如 span，因为要控制字体颜色）

- 该 dom 元素需设置 contenteditable="false" 属性，否则它是可以被修改的

[slide]

# <font color=#0099ff>如何保证前缀不会被删除并且禁止在前缀前面输入其他内容？</font>

- 监听输入事件

- 如何保证不会被删除：删除了再补回来

- 如何禁止输入：用户输入内容后，将输入内容删除

[slide]

# <font color=#ff9933>踩坑2：通过插入 html 的方法把删除的内容补回来</font>

光标位置不正确，只能寻找其他方法

[slide]

# <font color=#0099ff>什么是范围？</font>

定义：文档中的一个区域，而不必考虑界限（选择在后台完成，对用户是不可见的）

用处：在常规的 DOM 操作不能更有效地修改文档时，使用范围往往可以达到目的

详细内容可以参考《JavaScript 高级程序设计（第三版）》12.4 章节内容

[slide]

# <font color=#0099ff>创建一个范围</font>

- 创建一个范围：var range = document.creatRange();
- 通过光标位置得到一个范围：var range = window.getSelection().getRangeAt(0);

[slide]

# <font color=#0099ff>范围的常用属性</font>

- startContainer：包含范围起点的节点（即选区中第一个节点的父节点）
- startOffset：范围在 startContainer 中起点的偏移量
- endContainer：包含范围终点的节点（即选区中最后一个节点的父节点）
- endOffset：范围在 endContainer 中起点的偏移量
- collapsed：范围是否折叠（起点和终点是否在同一个位置）

[slide]

# <font color=#0099ff>范围的常用方法(一) - 简单选择</font>

- selectNode(node) 范围包括整个节点
- selectNodeContents(node) 范围只包括该节点的子节点
- setStartBefore(refNode) 将范围的起点设置在 refNode 之前
- setStartAfter(refNode) 将范围的起点设置在 refNode 之后
- setEndBefore(refNode) 将范围的终点设置在 refNode 之前
- setEndAfter(refNode) 将范围的终点设置在 refNode 之后

[slide]

# <font color=#0099ff>范围的常用方法(二) - 精确选择</font>

- setStart(node, index) 设置范围起始位置，node 会变成 startContainer ，index 是偏移量
- setEnd(node, index) 设置终点起始位置，node 会变成 endContainer ，index 是偏移量

[slide]

# <font color=#0099ff>范围的常用方法(三) - 操作范围中的内容</font>

- createContextualFragment(html) 将 html 片段转化为文档片段
- insertNode(node/fragment); 将节点或者片段插入到范围中
- deleteContents() 删除范围中的内容
- collapse(true/false) 折叠范围，true 将范围的终点移动到起点位置，false 将范围的起点移动到终点位置
- surroundContents 环绕范围插入内容（在富文本编辑器中很重要，本次需求没有用到该功能）

[slide]

# <font color=#0099ff>surroundContents 方法使用举例：修改字体颜色</font>

``` JavaScript
/* 获取鼠标选择的范围 */
var range = window.getSelection().getRangeAt(0);

/* 创建一个包围节点，将节点包围所选范围 */
var surroundNode = document.createElement('p');
surroundNode.style.color = 'red';
range.surroundContents(surroundNode);
```

[slide]

# <font color=#0099ff>如何将删除的内容补回来</font>

``` JavaScript
if (inputContent.indexOf(this.prefixSpan) < 0) {
  if (window.getSelection && window.getSelection().getRangeAt) {
    var range = window.getSelection().getRangeAt(0); /* 获取光标位置 */
    var fragment = range.createContextualFragment(this.prefixSpan); /* 将 html 片段转化为文档片段 */
    range.insertNode(fragment); /* 在光标位置插入该片段，光标范围会包括整个片段 */
    range.collapse(false); /* 折叠光标范围，true 折叠到插入节点最前面，false 到最后面 */
  }
}
```

[slide]

# <font color=#0099ff>禁止在前缀前输入内容</font>

``` JavaScript
if (inputContent.indexOf(this.prefixSpan) > 0) {
  var editorContainerDom = this.$editorContainer.get(0);
  var firstNode = editorContainerDom.firstChild;
  var prefixNode = editorContainerDom.getElementsByClassName('sms-editor-prefix')[0];
  if (document.createRange) {
    var range = document.createRange();
    range.setStart(firstNode, 0); /* 设置范围的起始位置为首节点第一个位置 */
    range.setEnd(prefixNode, 0); /* 设置范围的结束位置为前缀节点前一个位置 */
    range.deleteContents(); /* 删除这个范围的内容 */
  }
}
```

[slide]

# <font color=#0099ff>如何实现插入用户昵称？</font>

- 首先要解决的是往哪里插入的问题

- 解决了插入位置问题，再按照刚刚插入前缀的方法实现用户昵称的插入

[slide]

# <font color=#0099ff>记录插入位置</font>

- 初始化的时候默认将插入位置设置在编辑器最后（该逻辑要放在插入前缀之后）

- 在编辑器失焦或者键盘按起的时候记录光标最后的位置

``` JavaScript
/* 获得初始状态的位置 */
this.lastEditRange = this.getSafeRange();

/* 键盘按起和鼠标按起事件记录位置 */
this.$editorContainer.bind('keyup', function (e) {
  me.recordLastRange();
});
this.$editorContainer.bind('mouseup', function (e) {
  me.recordLastRange();
});

...

/* 获取安全的光标位置（即编辑框最后）*/
getSafeRange: function () {
  var newRange;
  if (document.createRange) {
    newRange = document.createRange();

    var editorContainer = this.$editorContainer.get(0);
    if (editorContainer.childNodes.length > 0) {
      newRange.setStartAfter(editorContainer.lastChild);
    } else {
      newRange.setStart(editorContainer, 0);
    }
  }
  return newRange;
},

/* 记录光标位置 */
recordLastRange: function() {
  if (window.getSelection && window.getSelection().getRangeAt) {
    this.lastEditRange = window.getSelection().getRangeAt(0);
  }
}
```

[slide]

# <font color=#0099ff>插入用户昵称</font>

``` JavaScript
this.$insertNicknameBtn.on('click', function (e) {
  e.preventDefault();
  me.insertNickname();
});

...

insertNickname: function() {
  var range = this.lastEditRange;

  if (range) {
    var fragment = range.createContextualFragment(this.nicknameSpan); /* 将 html 片段转化为文档片段 */
    range.insertNode(fragment); /* 在光标位置插入节点，光标范围会包括整个节点 */
    range.collapse(false); /* 折叠光标范围，true 移动到插入节点最前面，false 到最后面 */
  }
},
```

[slide]

# <font color=#ff9933>踩坑3：contenteditable="false" 元素虽不可编辑，但是阻止通过范围的方法插入内容 </font>

需要判断该位置是否可以插入用户昵称

[slide]

# <font color=#0099ff>如何判断该位置是否可以插入</font>

``` JavaScript
/* 判断该位置是否禁止插入或粘贴 */
isForbidden: function (range) {
  /* 光标范围在前缀签名节点、昵称节点内禁止插入或粘贴 */
  const forbiddenClassList = ['editor-prefix', 'editor-nickname'];
  if (forbiddenClassList.indexOf(range.startContainer.parentNode.className) > -1) {
    return true;
  }

  /* 光标起始点在前缀签名节点前禁止插入或粘贴 */
  return range.startContainer === this.$editorContainer.get(0) && range.startOffset === 0;
},
```

[slide]

# <font color=#ff9933>踩坑4：粘贴事件</font>

- 短信编辑框只能输入文字，但是可编辑 div 是可以容纳所有 dom 元素，包括图片和其他有样式的 div 元素，粘贴事件变得不可控

[slide]

# <font color=#0099ff>重写粘贴事件</font>

- 同样要注意在某些位置是无法插入的，原理和插入节点一样（如果不重写粘贴事件，是没有这个问题的）

``` JavaScript
pasteHandle: function (e) {
  var range;

  if (window.getSelection && window.getSelection().getRangeAt) {
    range = window.getSelection().getRangeAt(0);
  } else {
    return;
  }
  
  /* 在某些情况下禁止粘贴 */
  if (this.isForbidden(range)) {
    return;
  }

  var clipboardData = e.clipboardData || window.clipboardData;
  var pasteText = clipboardData.getData('text/plain');

  /* 过滤掉非文本内容 */
  if (!pasteText) {
    return;
  }

  /* 如果光标范围未折叠（即包含内容），删除掉包含的内容 */
  if (!range.collapsed) {
    range.deleteContents();
  }

  /* 粘贴剪切板内容 */
  var textNode = document.createTextNode(pasteText); /* 创建文本节点 */
  range.insertNode(textNode); /* 在光标位置插入文本节点，光标范围会包括整个节点 */
  range.collapse(false); /* 折叠光标范围，true 折叠到插入节点最前面，false 到最后面 */
},
```

[slide]

# <font color=#ff9933>踩坑5：聚焦后光标不显示</font>

- 现象：在不输入任何内容的情况下或者在文本最后插入用户昵称时，聚焦时如果鼠标不是落在有内容的最后一行，光标不会显示，此时无法输入任何内容

- 原因：如果编辑框最后子节点是个不可编辑（contenteditable="false"）节点，那么光标不会出现

[slide]

# <font color=#0099ff>聚焦后光标不显示的解决办法</font>

- 聚焦时判断：如果编辑框最后子节点不是文本节点，创建一个空的文本节点，并使光标移动到该位置

``` JavaScript
/* 显示光标 */
showSelection() {
  var editorDom = this.$editorContainer.get(0);
  if (editorDom.lastChild.nodeType !== 3) {
    /* 如果编辑框最后节点不是文本节点，添加一个空的文本节点 */
    var newTextNode = document.createTextNode('');
    editorDom.appendChild(newTextNode);
  }

  /* 如果编辑框最后节点是一个空的文本节点（不空光标自动会出现，不用再写这个逻辑），让光标出现；*/
  var lastChild = editorDom.lastChild;
  if (lastChild.nodeType === 3 && lastChild.wholeText === '') {
    var range = document.createRange();
    range.selectNode(lastChild);
    var selection = document.getSelection();
    selection.removeAllRanges();
    selection.addRange(range);
  }
},
```

[slide]

# <font color=#0099ff>总结</font>

- 签名前缀的实现（如何避免被删除，如何避免在前缀之前输入内容）

- 插入用户昵称的实现（如何定位要插入的位置，插入的时候要注意哪些位置是不能插入的）

- 粘贴功能的重写（过滤掉非文本内容，同样要注意粘贴的时候要注意哪些位置是不能粘贴的）

- 其他踩坑的地方（解决聚焦后光标不显示问题）

[slide]

# THE END

### THANK YOU
