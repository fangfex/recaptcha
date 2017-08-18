## 防刷组件（PC&WAP）用法说明




> 防刷组件（PC&WAP）通过前后端混合验证的算法实现了表单防刷的机制，验证算法涵盖了时间，空间（控件和鼠标坐标），页面元素，域名等方面，理论上该组件是安全有效的。调用该组件必须通过前后台的配合。

使用步骤：
1. 申请使用组件，须要发邮件提供相关信息，主要是为了记录相关信息，便于防刷的验证，相关信息包括表单名称，表单域名（建议提供三个，分别为本地开发、测试、正式。域名支持正式表达式，如`zu(.+?)\.fang\.com` ，可匹配`zu.bj.fang.com`,`zu.tj.fang.com`等），开发者（申请者），表单使用环境（pc,或wap）。
>  申请成功后，会分配一个key，通过key调用初始化接口。

2. 组件的初始化接口，初始化接口只能通过key调用，并且只能通过后台调用。并且可以设置简单模式、普通模式、严格模式，默认为简单模式。

```
https://apirecaptcha.fang.com/?c=index&a=codeInit&key={key}&mode=2
```

###### 接口说明：
该接口由客户端后台调用，负责生成一些初始化数据。

###### 接口参数

key | value | 说明
---|---|---
c | index | 不可修改
a | codeInit | 不可修改
key | 7d58e2808e8514580589fb3c09621a47 | 创建应用时分配
mode | 1或2或3，默认为1简单模式 | 1为简单模式，2为普通模式，3为严格模式

> 模式说明：验证算法有10项判断条件，简单模式下只要满足7项条件即可通过，普通模式要求10项条件全满足，包括10分钟内5次的限制，如果不满足10条，则会显示图片验证码，要求点击图片中的一个文字。严格模式下都会显示图片验证，要求点击图片中的两个文字。开发者可根据业务需要自主选择防刷模式。

示例代码：(php)

```php

function curl_get($url, $gzip=false){
    $curl = curl_init($url);
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_CONNECTTIMEOUT, 10);
    if($gzip) curl_setopt($curl, CURLOPT_ENCODING, "gzip"); // 关键在这里
    $content = curl_exec($curl);
    curl_close($curl);
    return $content;
}
$content = curl_get('https://apirecaptcha.fang.com/?c=index&a=codeInit&key=7d58e2808e8514580589fb3c09621a47&mode=2', true);
echo $content;
exit();
```

示例代码：(node)

```javascrpt

indexAction: function(req, res) {
    let params = {
        "c": "index",
        "a": "codeInit",
        "mode": 2,
        "key": checkcodeController.key,
        "init_time": new Date().getTime()
    };
    let reqData = qs.stringify(params);
    let request = http.request({
        hostname: 'apirecaptcha.fang.com',
        path: '/?' + reqData,
        method: 'get',
        encoding: null
    }, (response) => {
        let resData = '';
        let encoding = response.headers['content-encoding'];
        response.on('data', (chunk) => {
            if (encoding === 'gzip') {
                resData += zlib.gunzipSync(chunk);
            } else {
                resData += chunk;
            }

        });
        response.on('end', () => {
            res.send(resData);
        });
    });
    request.on('error', (e) => {
        res.send(e);
    });
    request.end();
},
```

3. 加载组件js文件。
```html
<script src="http://static.soufunimg.com/common_m/m_recaptcha/js/app.js" charset="uft-8"></script>
```

4. 调用组件初始化函数window.fCheck.init(options)，options是个Object，支持多个参数，有container容器选择器，url为本地ajax初始化接口，callback验证成功后的回调函数，width图片的宽度，height图片的高度，组合本身会有默认的宽高，如果不需要特别的处理，就需要设置width和height。

```javascript

(function(window, $) {
    // 调用验证控件
    window.fCheck.init({
        container: '.drag-content',
        url: "/act/?c=checkcode&a=index",
        callback: function() {
            // 验证成功后的回调
        }
    });
})(this, jQuery);

```
如果需要重新初始化，请调用`reinit`, 不需要传任何参数。

```javascript

(function(window, $) {
	// 重新初始化
	window.fCheck.reinit();
})(this, jQuery);

```
5. 表单提交，在表单提交时，需要将验证结果一并提交，用于后台校验。


```javascript
(function(window, $) {
	/**
	 * 表单提交
	 */
	$('.btn').on('click', function() {
		var username = $('input[name=username]');
		if (username.val() === ''){
			alert('请输入用户名');
			return false;
		}
		var password = $('input[name=password]');
		if (password.val() === ''){
			alert('请输入密码');
			return false;
		}
		// 判断是否操作了验证组件。
		if (window.fCheck.config.result === null){
			alert('验证码错误');
			return false;
		}
		// 将验证结果值添加的提交的数据中。
		var data = $.extend({username: username.val(), password: password.val()}, fCheck.config.result);
		// ajax post方式提交数据内容
		$.post('/act/?c=index&a=login', data, function(data, textStatus, xhr) {
			if (data.code === '100') {
				alert('验证通过，提交成功！')
			}
		});
	});

})(this, jQuery);

```
或者，直接添加到表单中，进行提交。
```javascript
// 获取验证结果
var result = window.fCheck.config.result;
var form = $('#formId');
// 将验证结果值以隐藏域的形式添加的提交的表单中。
for(var p in result){
	form.append('<input name="'+p+'" type="hidden" value="'+result[p]+'" >')
}
// 提交表单
form.submit();
```
6. 在表单数据处理后台，调用组件验证接口进行有效性检验。

###### 接口示例：
```
http://apirecaptcha.fang.com/?c=index&a=codeCheck&key={key}&gt={gt}&challenge={challenge}&validate={validate}
```
###### 接口说明：
该接口由客户端后台调用，验证组件相关值的一致性。

###### 接口参数

key | value | 说明
---|---|---
c | index | 不可修改
a | codeCheck | 不可修改
key | 7d58e2808e8514580589fb3c09621a47 | 创建应用时分配
gt | 7d58e2808e8514580589fb3c09621a47 | 应用ID
challenge | 7d58e2808e8514580589fb3c09621a47 | 当前实例ID
validate | 000e177ab1a3e873f5e904b5df60b610 | 验证码

###### 接口返回值

```json
{
    code: "100",
    message: "successed"
}
```

key | value | 数据类型 | 说明
---|:---|:---|:---
code | 100或101或102 | string| 代号,100表示验证通过，101表示不通过，102表示时需要参照used。
message | successed或failed | string | 描述
used | 数字，例如1 | int | 验证次数（从0开始的数字）

示例代码：(php)

```php
function curl_get($url, $gzip=false){
    $curl = curl_init($url);
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_CONNECTTIMEOUT, 10);
    if($gzip) curl_setopt($curl, CURLOPT_ENCODING, "gzip"); // 关键在这里
    $content = curl_exec($curl);
    curl_close($curl);
    return $content;
}
$gt = $_GET['fc_gt'];
$challenge = $_GET['fc_challenge'];
$validate = $_GET['fc_validate'];
$content = curl_get('https://apirecaptcha.fang.com/?c=index&a=codeCheck&key='+$key+'&gt='+$gt+'&challenge='+$challenge+'&validate='+$validate+'', true);
$result = json_decode($content);
if ($result.code == '100') {
    // 验证成功
    // 将表单数据写入数据库。
} else {
    // 验证失败
}
echo $content;
exit();
```
示例代码：(node)

```javascript
loginAction: function(req, res) {
    let query = req.body;
    let params = {
        "c": "index",
        "a": "codeCheck",
        "key": checkcodeController.key,
        "gt": query.fc_gt,
        "challenge": query.fc_challenge,
        "validate": query.fc_validate
    };

    let reqData = qs.stringify(params);;
    let request = http.request({
        hostname: 'apirecaptcha.fang.com',
        path: '/?' + reqData,
        method: 'get',
        encoding: null
    }, (response) => {
        let resData = '';
        let encoding = response.headers['content-encoding'];
        response.on('data', (chunk) => {
            if (encoding === 'gzip') {
                resData += zlib.gunzipSync(chunk);
            } else {
                resData += chunk;
            }

        });
        response.on('end', () => {
            res.send(resData);
        });
    });
    request.on('error', (e) => {
        res.send(e);
    });
    request.end();
}
```

#### [附]. PC在线演示示例：

[http://activities.m.fang.com/recaptcha/pc.html](http://activities.m.fang.com/recaptcha/pc.html)


#### [附]. WAP在线演示示例
[http://activities.m.fang.com/recaptcha/wap.html](http://activities.m.fang.com/recaptcha/wap.html)

![WAP在线演示示例](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAMcAAADJCAYAAACXDya2AAAgAElEQVR4Ae2dC9Ru1fT/n5LuF5RQueWgUqOIQbr8OoqRUYkiTpR0EUV0vxh0IockUlEkl3Lpoo5GV92lEgkdieOSCEXKNUS8//F5/mfuvvv7zP0++7m8qfddc4z33XvPPddcc8211n7WXHOutRabmJiY6BQoGiga6NHA4j2YgigaKBroaqB0jtIQigYaNFA6R4NiCrpooHSO0gaKBho0UDpHg2IKumhgiUwFRx55ZA/66quv7jz2sY/trL/++tU7cGuuuWbnKU95Sg33/Oc/v7P88st3cf/+97871113XWfzzTevaH7/+993br311hruxz/+ceeuu+6q4SLBEUccEbdjvf7yl7/sfO5zn0t59ssz0xGM1l577c6OO+5Y8czo0NsLX/jCzjLLLFPRcdMvzxrxoocPfOADnX/96189r1ZfffXOHnvsUeGb5Nhggw06j3nMY7p0TFx+/etfr9XBPffc0/nBD35Qw/3sZz/r/PrXv67hIiMtQ1OeT3jCE7p6ijToA72BDwC38cYbdx796Ed3UX/729863/nOd2p53nHHHZ2f//znNdyCBQs69957bw0XPFW2wE16ZSrXYe7cuUzv1v6WWGKJ2jPvH/WoR/XgFltssR6c8+J58cUX76HL8thnn31cvLE933777RPLL798jxzPe97z+uZx0EEH9aRbccUVJ84888xa2lmzZvXQZfpYdtlla+naPsybN6+nHpZbbrmJU045pcaCMnm+besqo8vqfrfddqvlSd15nlkdZ23B0/HcVo4sD9r0oFCGVZN+OsrLmayB0jlmcu2Xsk+qgdI5JlVPeTmTNZAa5BhDSyxRf/Wf//ynaxwtvfTSlb7+/ve/d5ZddtnOox71qAqH4YQxvthii3VxGHngVlhhhYrmv//9b+cf//hHDXf//fd3MN4932uuuaZKN8jNq171qpT8K1/5SiXvb37zm84///nPWp7IiwGq6THc77777g4TDQE//OEPO0sttVRnySWXDFQHfXz4wx/ufPnLX65wv/jFL7r5hT548cADD3SWW265zuKL//9vE3miD83zT3/6U2fhwoVdwz2YIS+GMMZ8wPXXX98hveoNXX7jG9+oGeTQax3w/Ne//jWVTemoK8qluKa6uvbaa0Os7pW6U7lA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)
