# 博客搭建


&lt;!--more--&gt;

## 前置

下载好Git、Go 和 Dart Sass、hugo

Dart Sass我这用的npm下的

```
npm install -g sass
sass --version                 //验证安装
```

hugo下载：https://github.com/gohugoio/hugo/releases（下载extend拓展版的，方便自定义加东西，记得加环境变量）

其他我都配好了，不多说

## 博客配置

然后按照这个来吧：

https://fixit.lruihao.cn/zh-cn/documentation/getting-started/quick-start/

## 美化记录（持续更新ing）

### 2025.2.28

有人催我加友链，于是弄下友链，感觉 loveit 主题和 fixit 主题差不多，按照文章：https://blog.233so.com/2020/04/friend-link-shortcodes-for-hugo-loveit/ 来就好，只需要注意下在修改`FixIt/assets/css/_page/`的`_single.scss`时引入的行应该为

```
@import &#34;../_partials/_single/friend&#34;;
```

fixit 的目录比 loveit 的目录少了个 s ，其他的话叫 ai 改了个颜色，把旋转去掉了，再加了个微微放大的动画，我的 friend.scss

```
// ===== 变量定义 =====
$shadow-green: rgba(50, 205, 50, 0.3);    // 绿色阴影
$hover-green: rgba(144, 238, 144, 0.3);   // 悬停背景色
$hover-scale: 1.02;                       // 悬停放大比例
$transition-time: 0.3s;                   // 过渡动画时间

// ===== 基础样式 =====
.friendurl {
  text-decoration: none !important;
  color: black;
}

.myfriend {
  width: 56px !important;
  height: 56px !important;
  border-radius: 50%;
  border: 1px solid #ddd;
  padding: 2px;
  box-shadow: 1px 1px 1px rgba(0, 0, 0, 0.15);
  margin: 14px 0 0 14px !important;
  background-color: #fff;
}

.frienddiv {
  // ===== 布局属性 =====
  height: 92px;
  margin-top: 10px;
  width: 48%;
  display: inline-block !important;
  border-radius: 5px;
  
  // ===== 视觉效果 =====
  background: rgba(255, 255, 255, 0.2);
  box-shadow: 4px 4px 2px 1px $shadow-green;
  
  // ===== 过渡动画 =====
  transition: all $transition-time ease-in-out;

  // ===== 悬停状态 =====
  &amp;:hover {
    background: $hover-green;
    transform: scale($hover-scale);
    box-shadow: 4px 4px 6px 2px rgba(50, 205, 50, 0.25); // 加强阴影
  }

  // ===== 左侧头像区域 =====
  .frienddivleft {
    width: 92px;
    float: left;
    margin-right: 2px;
  }

  // ===== 右侧文字区域 =====
  .frienddivright {
    margin-top: 18px;
    margin-right: 18px;
  }

  // ===== 文字控制 =====
  .friendname, .friendinfo {
    text-overflow: ellipsis;
    overflow: hidden;
    white-space: nowrap;
  }
}

// ===== 手机端适配 =====
@media screen and (max-width: 600px) {
  .frienddiv {
    width: 100% !important;  // 单列显示
    
    &amp;:hover {
      transform: scale(1.01); // 缩小放大比例
    }
    
    .friendinfo {
      display: none;
    }
    
    .frienddivleft {
      width: 84px;
      margin: auto;
    }
    
    .frienddivright {
      height: 100%;
      margin: auto;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    
    .friendname {
      font-size: 14px;
    }
  }
}
```



---

> 作者: 6s6  
> URL: http://localhost:1313/posts/5c6225e/  

