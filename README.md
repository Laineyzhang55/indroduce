# 三分钟PPT
## 项目背景
之前的工作中经常需要做PPT，为了能够做出一个简介大方又不失美观的PPT常常需要耗费大量的时间，而且学习PPT的制作也绝非一日之功，快速完成一个PPT就是本项目的最终目的。
## 项目调研
通过搜索js是实现的slides框架，发现排名第一的就是reveal.js。reveal.js提供了一套完成的体系去制作PPT，包括各种主题，切页效果，甚至支持Markdown语法。但是通过官网使用体验发现学习成本同样不低，下载源码使用的话仅限于技术人员可以使用，使用面太小。因此在reveal.js的基础上，我们进一步进行加工，使之可以在线快速完成PPT制作，无需写代码，也无需一页一页的编辑。

## 项目介绍
使用原生Javascript实现一款简洁高效的在线ppt制作平台。使用该工具用户可使用markdown在数分钟完成ppt制作，支持内容编辑、主题切换、特效切换、下载、演讲者模式等功能。

## 项目实施
- 网页设计
> 分为主页和菜单页，菜单页采用了tabs的方法设计样式，主要有编辑、主题设置、打印、阅读者模式和使用说明5个功能区。

```html
<div class="detail">
  <div class="tab">内容编辑</div>
  <div class="tab">主题与特效</div>
  <div class="tab">PDF下载</div>
  <div class="tab">演讲者模式</div>
  <div class="tab">使用说明</div>
</div>
<!--相同数量的content，和tab一一对应-->
<div class="content">
  .....
</div>
.....
```

- 完善编辑板块的功能
  1. 观察reveal.js的编辑方式，发现是通过`<section>`的数量来区分页面的，而页面的主次是通过“#”的数量。一个“#”开头是主页，两个是次页，三个是次页的子页面，超过三个就是正常的Markdown的标题的语法。
  使用正则表达式来定义好分页的规则：

  定义判断主页面和子页面的函数
  ```js
  const isMain = str => (/^#{1,2}(?!#)/).test(str)
  const isSub = str => (/^#{3}(?!#)/).test(str)
  ```

  定义规则，如果是主页面添加reveal.js规定的`<section>`和`<textarea>`标签，如果有子页面多添加一层`<section>`包裹。

  ```js
  function convert(raw) {
  let arr = raw.split(/\n(?=\s*#{1,3}[^=#])/).filter(s => s!='').map(s => s.trim())
  let html = ''
  for(let i=0; i<arr.length; i++) {
    if(arr[i+1] !== undefined){
      if(isMain(arr[i]) && isMain(arr[i+1])) {
       html += `
  <section data-markdown>
    <textarea data-template>
      ${arr[i]}
    </textarea>
  </section>
  `
      } else if(isMain(arr[i]) && isSub(arr[i+1])) {
   html += `
  <section>
  <section data-markdown>
    <textarea data-template>
      ${arr[i]}
    </textarea>
  </section>
  `
      }else if(isSub(arr[i]) && isSub(arr[i+1])) {
  html += `
  <section data-markdown>
    <textarea data-template>
      ${arr[i]}
    </textarea>
  </section>
  `
      }else if(isSub(arr[i]) && isMain(arr[i+1])){
     html += `<section data-markdown>
        <textarea data-template>
          ${arr[i]}
        </textarea>
      </section>
    </section>
  `
      }
  }else {
      if(isMain(arr[i])) {
          html += `
  <section data-markdown>
    <textarea data-template>
      ${arr[i]}
    </textarea>
  </section>
  `
      } else if(isSub(arr[i])) {
  html += `<section data-markdown>
        <textarea data-template>
          ${arr[i]}
        </textarea>
      </section>
    </section>
  `
        }
      }
    } 
  return html
  }
  ```
  
2. 设置内容输入框，将输入的内容通过上一步定义的规则进行判断，并转换成HTML插入到`[class="slide"]`的DOM内。

```js
//Menue中的方法
//使用localStorage使得编辑的内容得以保存，当重新加载后可以保留上一次编辑的内容，方便修改
  bind() {
    this.$saveBtn.onclick = () => {
      localStorage.markdown = this.$editInput.value
      location.reload()
    }
  },

  start() {
    this.$editInput.value = this.markdown
    this.$slideContainer.innerHTML = convert(this.markdown)
    //initialize是reveal.js提供的初始化方法
    Reveal.initialize({
          controls: true,
          progress: true,
          center: localStorage.aligh === 'left-top' ? false : true,
          hash: true,
          transition: localStorage.transition || 'slide', // none/fade/slide/convex/concave/zoom
          // More info https://github.com/hakimel/reveal.js#dependencies
          .....
        })
  }

```
3. 编辑主题切换
reveal.js设置文字的对齐方式由下面代码的`center`控制，值为`true`时居中，否则左上对齐；
切页效果由`transition`取值决定；

```js
Reveal.initialize({
          controls: true,
          progress: true,
          center: localStorage.align === 'left-top' ? false : true,
          hash: true,
          transition: localStorage.transition || 'slide', // none/fade/slide/convex/concave/zoom
          ....
)}
```

因此为了让这两个属性的取值被用户在页面的选择取值决定，同样将用户选择的值保存在localStorage中，再通过localStorage调用。

```js
//$transition和$align都是页面的DOM节点
  this.$transition.onchange = function() {
      localStorage.transition = this.value
      location.reload()
    }

    this.$align.onchange = function() {
      localStorage.align = this.value
      location.reload()
    }

```

主题的变化是通过引入不同主题的CSS文件控制，因此可以在用户选择了主题后创建一个新的`<link>`标签插入到`head`中。

```js
setTheme(theme) {
    localStorage.theme = theme
    location.reload()
  },

  loadTheme() {
    let theme = localStorage.theme || 'sky'
    let $link = document.createElement('link')
    $link.rel ='stylesheet'
    $link.href = `./css/theme/${theme}.css`
    document.head.appendChild($link)

    Array.from(this.$$figures).find($figure => $figure.dataset.theme === theme).classList.add('select')
    this.$transition.value = localStorage.transition || 'slide'
    this.$align.value = localStorage.align || 'center'
    this.$reveal.classList.add(this.$align.value)
  }
```
4. 设置打印模式

> reveal.js提供的打印方法是在url中加上`?print-pdf#/`，因此可以在点击打印时创建一个`href`值符合的a链接，并在新页面打开。

```js

   bind() {
    this.$print.addEventListener('click', () =>{
      let $link = document.createElement('a')
      $link.setAttribute('target', '_blank')
      $link.setAttribute('href', location.href.replace(/#\/.*/, '?print-pdf'))
      $link.click()
    })
  },

```

5. 完善上传图片功能
> 基于LeanCloud的存储功能完善PPT的图片功能。主要理念是监听[type=file]的改变，调用LeanCloud的接口上传图片，上传成功后在编辑框内展示Markdown的图片语法`![filename](url)`。

初始化：选中上传按钮以及编辑框，绑定LeanCloud的应用

```js
init(){
    this.$fileInput = $('#image-uploader')
    this.$textarea = $('.editor textarea')

    AV.init({
      appId: "lrq81oYdxLjnI8EcPdR5QvPW-gzGzoHsz",
      appKey: "Stl9ULTfyXhA6xTgnw03eFKl",
      serverURL: "https://lrq81oyd.example.com"
    })
```

文件上传逻辑：第一，必须是图片格式；第二，图片大小限制。

```js
let self = this
  this.$fileInput.onchange=function(){
    if(this.files.length > 0) {
      let localFile = this.files[0]
      if(localFile.size/1048576 > 2) {
        alert('文件不能超过2M')
        return
      }
      self.insertText(`![上传中，进度0%]()`)
      let  avFile = new AV.File(encodeURI(localFile.name), localFile)
      avFile.save({ 
        keepFileName: true, 
        onprogress(progress) {
          self.insertText(`![上传中，进度${progress.percent}%]()`)
        }
      }).then(file => {
        let text = `![${file.attributes.name}](${file.attributes.url}?imageView2/0/w/800/h/400)`
        self.insertText(text)
      }).catch(err => console.log(err))
    } 
  }
```

上列代码中的`insertText`是设置的编辑框内的展示逻辑：
> 开始上传-上传结束：展示`![上传中，进度${progress.percent}%]()`；
完成上传：展示`![${file.attributes.name}](${file.attributes.url}?imageView2/0/w/800/h/400)`

使用到了`textarea`的API`selectionStart`、`selectionEnd`、`focus`以及字符串截图的API`substring`。

```js
insertText(text = '') {
    let $textarea = this.$textarea
    let start = $textarea.selectionStart
    let end = $textarea.selectionEnd
    let oldText = $textarea.value

    $textarea.value = `${oldText.substring(0, start)}${text} ${oldText.substring(end)}`
    $textarea.focus()
    $textarea.setSelectionRange(start, start + text.length) 
  }
```
