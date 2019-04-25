# 代码规范


- [Airbnb](https://github.com/airbnb/javascript)
- [看云](https://www.kancloud.cn/kancloud/javascript-style-guide)
- [Airbnb JavaScript 代码规范(ES6)](https://www.kancloud.cn/kancloud/javascript-style-guide/43119)
- [NetEase](http://nec.netease.com/standard/css-practice.html)
- [CSS BEM 书写规范](https://github.com/Tencent/tmt-workflow/wiki)
- [Bootstrap 编码规范](https://codeguide.bootcss.com)

-----

- [Jquery代码组织方法优化](https://blog.csdn.net/qq_36370731/article/details/82819968)
- [使用jQuery时如何更好的组织代码？](https://www.zhihu.com/question/26348002/answer/32617352)


```js

var index = {
    // 初始化
    init: function () {
        var self = this
        self.target()
        self.created()
        self.render()
    },
 
    // 相关DOM节点存储
    target: function () {
        var self = this;
        self.barItem = $('.bar-item') // banner切换按钮 
    },
 
    // 页面初始化数据获取
    created: function () {
        var self = this;
    },
 
    // 事件绑定
    render: function () {
        var self = this
        // 切换banner事件
        self.barItem.click(function () {
            $(this).attr('class', "bar-item bar-cur").siblings().attr('class', "bar-item")
            $(".banner a").eq($(this).attr("index")).fadeIn(1800).siblings().fadeOut();
        })
    }
}
 
$(function () {
    index.init();
});
```


```js
(function() {
  var modules = {};

  modules.bootstrapFileInput = function() {
  };

  modules.inputCheck = function() {
  };

  modules.imagePreview = function() {
  };

  modules.jQueryFormData = function() {
  };

  $(function() {
    modules.bootstrapFileInput();
    modules.inputCheck();
    modules.imagePreview();
    modules.jQueryFormData();
  });

}).call(this);
```