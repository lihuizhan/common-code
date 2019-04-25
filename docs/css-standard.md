# CSS规范

- [Airbnb](https://github.com/airbnb/javascript)
- [看云](https://www.kancloud.cn/kancloud/javascript-style-guide)
- [Airbnb JavaScript 代码规范(ES6)](https://www.kancloud.cn/kancloud/javascript-style-guide/43119)
- [NetEase](http://nec.netease.com/standard/css-practice.html)
- [CSS BEM 书写规范](https://github.com/Tencent/tmt-workflow/wiki)
- [Bootstrap 编码规范](https://codeguide.bootcss.com)

### CSS书写顺序
  
1. 位置属性(position, top, right, z-index,display, float等)  
2. 大小(width, height, padding, margin)  
3. 文字系列(font, line-height, letter-spacing,color- text-align等)  
4. 背景(background, border等)  
5. 其他(animation, transition等)  

```css
.declaration-order {
  /* Positioning */
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: 100;

  /* Box-model */
  display: block;
  float: right;
  width: 100px;
  height: 100px;

  /* Typography */
  font: normal 13px "Helvetica Neue", sans-serif;
  line-height: 1.5;
  color: #333;
  text-align: center;

  /* Visual */
  background-color: #f5f5f5;
  border: 1px solid #e5e5e5;
  border-radius: 3px;

  /* Misc */
  opacity: 1;
}
``` 

显示属性 | 自身属性 | 文本属性和其他修饰
|---|---|---
display | width | font
visibility  | height | text-align
position | margin | text-decoration
float | padding | vertical-align
clear | border | white-space
list-style | overflow | color
top | min-width | background



### [组织项目文件目录结构 – 非常规文件](https://www.jianshu.com/p/3a7bcfdff060)

---------

- robots.txt
- favicon.ico
- humans.txt
- .editorconfig
- .gitignore
- LICENSE.txt
- README.md
- CHANGLOG.md

[^CSS规范 - 最佳实践 - 恋空一译]: https://mrlhz.github.io/2018/10/20/css-practice/