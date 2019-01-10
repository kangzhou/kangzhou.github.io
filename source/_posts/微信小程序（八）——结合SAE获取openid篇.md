---
title: 微信小程序（八）——结合SAE获取openid篇
date: 2018-01-18
categories: "Android"
tags: "小程序"
---
# 概述
我们在前端微信小程序发起登录请求后，成功会返回一个code：用户登录凭证（有效期五分钟）。开发者需要在开发者服务器后台调用 api，使用 code 换取 openid 和 session_key 等信息。
而且这个服务器必须要有证书的验证。没有证书的话不能使用本地的服务器，这就要我们再搭建一个新浪云服务SAE。
<!--more-->
# 搭建SAE
首先用微博账号登录（没有的话可以用手机注册）进去。如果是第一次使用的话要进行实名认证（一到两天的时间）。
接着进入SAE控制台，创建新应用：
![](http://oxr4g4c3v.bkt.clouddn.com/sae3.jpg)
如果要使用免费版的话呢，可以像我这样设置，二级域名可以以个人名填写：
![](http://oxr4g4c3v.bkt.clouddn.com/sae4.jpg)
然后结合刚刚设置的二级域名，我们可以直接在浏览器，访问我们的服务器了：
![](http://oxr4g4c3v.bkt.clouddn.com/sae5.jpg)
之后我们可以上传ZIP代码包，或者编辑代码:
![](http://oxr4g4c3v.bkt.clouddn.com/sae1.jpg)
编辑代码：
![](http://oxr4g4c3v.bkt.clouddn.com/sae2.jpg)

# 编辑代码
可以把以下代码全部复制到SAE中去
```php
$code = $_POST["code"];//POST请求，获取小程序发来的code值
$appid = "你的小程序APPID";
$secret = "你的小程序密钥";
$api = "https://api.weixin.qq.com/sns/jscode2session?appid={$appid}&secret={$secret}&js_code={$code}&grant_type=authorization_code";//拼接参数
 function httpGet($url) {
    $curl = curl_init();
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($curl, CURLOPT_TIMEOUT, 500);
    // 为保证第三方服务器与微信服务器之间数据传输的安全性，所有微信接口采用https方式调用，必须使用下面2行代码打开ssl安全校验。
    // 如果在部署过程中代码在此处验证失败，请到 http://curl.haxx.se/ca/cacert.pem 下载新的证书判别文件。
    curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, true);
    curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, true);//ssl验证
    curl_setopt($curl, CURLOPT_URL, $url);
    $res = curl_exec($curl);
    curl_close($curl);
    return $res;
}
$str = httpGet($api);//将openid返回给前端
echo $str
```
如果你的文件名为logincode.php，那么POST请求的路径就是：你的服务器域名+文件名。
例如我的：https://1.kanging01.applinzi.com/logincode.php 。
小程序代码：
```html
// 登录初始化
    wx.login({
	   // 发送 res.code 到后台换取 openId, sessionKey, unionId
        wx.request({
          url: 'https://1.kanging01.applinzi.com/logincode.php',
          data: {
            code: res.code//发送code
          },
          method: "POST",//post请求
          header: {
            'content-type': 'application/x-www-form-urlencoded' // 默认值
          },
          success: function (res) {//包含openid的res
            console.log( res)
          },
          fail: function (err) {
            console.log("服务器访问失败01")
          }
	  }
    })
	```
# 模板消息
顺带提一下，模板消息也类似获取openid，不同的是，需要在公众者平台选用设置一个模板，再结合SAE发送模板消息：
![](http://oxr4g4c3v.bkt.clouddn.com/mubanmsg.jpg)
**resultmuban.php:**
```php
    //获取参数
	$openid = $_GET["openid"];//
	$username = $_GET["username"];//
	$address = $_GET["address"];
	$form_id = $_GET["form_id"];//表单提交，小程序会返回的formid
	$type = $_GET["types"];//请假还是出差
	$st = $_GET["st"];//开始时间
	$et = $_GET["et"];//结束时间
	$checkname = $_GET["checkname"];//初审人
	$sendnum = $_GET["sendnum"];
	$fristid = $_GET["fristid"];//初审人openid
	$secondid = $_GET["secondid"];//终审人openid
	
	$appid = "你的APPID";
    $secret = "你的密钥";
	if($type=="出差"){//拼凑出差模板json
    //template_id就是模板的id，touser就是接收人openid
    $data = <<<END
{
  "touser": "{$fristid}",  
  "template_id": "sLW6c-_27dJMtB8rVFhGzSOEEjo_93opN2pUiaj8_Hs", 
  "page": "/pages/index/index",          
  "form_id": "{$form_id}",         
  "data": {
      "keyword1": {
          "value": "{$username}", 
          "color": "#173177"
      }, 
      "keyword2": {
          "value": "{$type}",  
          "color": "#173177"
      }, 
      "keyword3": {
          "value": "{$address}",  
          "color": "#173177"
      }, 
      "keyword4": {
          "value": "{$st}", 
          "color": "#173177"
      },
	  "keyword5": {
          "value": "{$et}", 
          "color": "#173177"
      },
	  "keyword6": {
          "value": "{$checkname}", 
          "color": "#173177"
      } 
  },
  "emphasis_keyword": "keyword1.DATA" 
}
END;
        
        $data2 = <<<END
{
  "touser": "{$secondid}",  
  "template_id": "sLW6c-_27dJMtB8rVFhGzSOEEjo_93opN2pUiaj8_Hs", 
  "page": "/pages/index/index",          
  "form_id": "{$form_id}",         
  "data": {
      "keyword1": {
          "value": "{$username}", 
          "color": "#173177"
      }, 
      "keyword2": {
          "value": "{$type}",  
          "color": "#173177"
      }, 
      "keyword3": {
          "value": "{$address}",  
          "color": "#173177"
      }, 
      "keyword4": {
          "value": "{$st}", 
          "color": "#173177"
      },
	  "keyword5": {
          "value": "{$et}", 
          "color": "#173177"
      },
	  "keyword6": {
          "value": "{$checkname}", 
          "color": "#173177"
      } 
  },
  "emphasis_keyword": "keyword1.DATA" 
}
END;
        }else{//拼接请假json
            $data = <<<END
{
  "touser": "{$fristid}",  
  "template_id": "sLW6c-_27dJMtB8rVFhGzU7S4IY9SvRTx-r7XAqrmdo", 
  "page": "/pages/index/index",          
  "form_id": "{$form_id}",         
  "data": {
      "keyword1": {
          "value": "{$username}", 
          "color": "#173177"
      }, 
      "keyword2": {
          "value": "{$type}",  
          "color": "#173177"
      }, 
      "keyword3": {
          "value": "{$st}", 
          "color": "#173177"
      },
	  "keyword4": {
          "value": "{$et}", 
          "color": "#173177"
      },
	  "keyword5": {
          "value": "{$checkname}", 
          "color": "#173177"
      } 
  },
  "emphasis_keyword": "keyword1.DATA" 
}
END;
        $data2 = <<<END
{
  "touser": "{$secondid}",  
  "template_id": "sLW6c-_27dJMtB8rVFhGzU7S4IY9SvRTx-r7XAqrmdo", 
  "page": "/pages/index/index",          
  "form_id": "{$form_id}",         
  "data": {
      "keyword1": {
          "value": "{$username}", 
          "color": "#173177"
      }, 
      "keyword2": {
          "value": "{$type}",  
          "color": "#173177"
      }, 
      "keyword3": {
          "value": "{$st}", 
          "color": "#173177"
      },
	  "keyword4": {
          "value": "{$et}", 
          "color": "#173177"
      },
	  "keyword5": {
          "value": "{$checkname}", 
          "color": "#173177"
      } 
  },
  "emphasis_keyword": "keyword1.DATA" 
}
END;
    }
	//要获取token，拼接url
    $gettoken_url = "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid={$appid}&secret={$secret}";
        function httpPost($data,$url){
            $ch = curl_init();
            curl_setopt($ch, CURLOPT_URL, $url);
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
            curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE);
            curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
            curl_setopt($ch, CURLOPT_USERAGENT, 'Mozilla/5.0 (compatible; MSIE 5.01; Windows NT 5.0)');
            curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
            curl_setopt($ch, CURLOPT_AUTOREFERER, 1);
            curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
            $tmpInfo = curl_exec($ch);
            if (curl_errno($ch)) {
                return curl_error($ch);
            }
            curl_close($ch);
            return $tmpInfo;
        }
        function httpGet($url) {
            $curl = curl_init();
            curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
            curl_setopt($curl, CURLOPT_TIMEOUT, 500);
            // 为保证第三方服务器与微信服务器之间数据传输的安全性，所有微信接口采用https方式调用，必须使用下面2行代码打开ssl安全校验。
            // 如果在部署过程中代码在此处验证失败，请到 http://curl.haxx.se/ca/cacert.pem 下载新的证书判别文件。
            curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, true);
            curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, true);
            curl_setopt($curl, CURLOPT_URL, $url);
            $res = curl_exec($curl);
            curl_close($curl);
            return $res;
        }
        $retoken = httpGet($gettoken_url);
        $arr = json_decode($retoken,true);
        $token = $arr["access_token"];
        $sendTokenApi = "https://api.weixin.qq.com/cgi-bin/message/wxopen/template/send?access_token={$token}";
        $res = httpPost($data,$sendTokenApi);//发送模板消息
		
        if($secondid!=""){//如果审核人有两个的话，就再发一条模板消息
           $res = httpPost($data2,$sendTokenApi);
        }
        echo $res//发送消息后的结果，成功还是失败返回
```
这消息是由小程序发起的（POST请求），将数据发送到服务器，服务器再将消息拼接发送给接收人的手机上，类似一条微信消息，提示用户结果。

# 总结
到此这个小程序的大致过程已经介绍了一遍，可以看出，小程序的开发过程并不是很困难。反而做后台就挺复杂的，包括设计数据库，后台逻辑，高并发处理等等，这里不包括php语言本身。总之小程序还是值得学习一下，未来的互联网潮流应该还是由腾讯这只大佬来引领，马云爸爸坚持住。

# 下载
参看{% post_link 微信小程序（九）——资源下载 %} 
