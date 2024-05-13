<p align="center" style="color: red;font-size: 20px;" > 警告：滥用这些API可能会造成不可预测的后果，本研究均在根据代码复现的环境中进行，请谨慎使用。您使用此代码的一切行为与本人无关，如果造成不良结果请您后果自负。</p>
<p align="center" style="font-size: 15px;" >校园网认证系统经常更新，下面方法不一定随时可用</p>

## 1. 登录认证
|var|解释|示例|
|---|----|---|
|identity|网络登录账号(学号)|B11451419|
|password|登陆密码|1919810!senpai|
|myip|你要登陆的ip|10.114.51.4|
|operator|你的运营商:cmcc,njxy,(null)|cmcc|

 其中cmcc是移动，njxy是电信，空是校园网缴费的

### Code
可以直接填好下面的链接浏览器打开， `${myip}` 为空时可以登陆本机
`https://p.njupt.edu.cn:802/eportal/portal/login?callback=dr1003&login_method=1&user_account=%2C0%2C'${identity}'%40'${operator}'&user_password='${password}'&wlan_user_ip='${myip}'&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac`
#### 最简单版本Bash脚本
```bash
#!/bin/bash
identity=''
operator=''
password=''
curl -k -s 'https://p.njupt.edu.cn:802/eportal/portal/login?callback=dr1003&login_method=1&user_account=%2C0%2C'${identity}'%40'${operator}'&user_password='${password}'&wlan_user_ip='`curl -k -s "https://10.10.244.11/a79.htm" | sed -nE 's/.*ss5="([0-9\.]*?)".*/\1/p'`'&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac'
```

#### 全功能，自动登陆本机，客观上改变myip为任何值则为登陆您输入的ip
```bash
#!/bin/bash
myip=$(curl -k -s "https://10.10.244.11/a79.htm" | sed -nE 's/.*ss5="([0-9\.]*?)".*/\1/p') #nullable if you cannot obtain your ip
identity=''
operator=''
password=''

function login() {
result=$(curl -k -s 'https://10.10.244.11:802/eportal/portal/login?callback=dr1003&login_method=1&user_account=%2C0%2C'${identity}'%40'${operator}'&user_password='${password}'&wlan_user_ip='${myip}'&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac' \
  -H 'cookie: program=2; vlan=0; ssid=null; areaID=null; ip='${myip}' '  )
# Check the result
if [[ $result =~ "\"ret_code\":2" ]]; then
  echo "already logged in"
elif [[ $result =~ "Portal协议认证成功" ]]; then
  echo "$myip successfully logged in"
else
  echo "$result"
fi
}

function login_dbg() {
  echo your ip: $myip
  curl -k 'https://10.10.244.11:802/eportal/portal/login?callback=dr1003&login_method=1&user_account=%2C0%2C'${identity}'%40'${operator}'&user_password='${password}'&wlan_user_ip='${myip}'&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac' \
  -H 'cookie: program=2; vlan=0; ssid=null; areaID=null; ip='${myip}' ' \
  -v
}

if [ "$1" != "-v" ]; then
    login
else
    login_dbg
fi
```

#### Powershell Desktop (5.1) 全功能
如果无法运行，请管理员权限PowerShell下
```Powershell
Set-ExecutionPolicy RemoteSigned
```
```PowerShell
#requires -PSEdition Desktop

# 学号，运营商，密码
$identity = ""
$operator = ""
$password = ""
#  解析结果，解决各种dns疑难杂症
$target_ip = "10.10.244.11"
function login() {
    # 解决ip下SSL证书问题
    add-type @"
using System.Net;
using System.Security.Cryptography.X509Certificates;
public class TrustAllCertsPolicy : ICertificatePolicy {
    public bool CheckValidationResult(
        ServicePoint srvPoint, X509Certificate certificate,
        WebRequest request, int certificateProblem) {
        return true;
    }
}
"@
[System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy

    # 获取ip
    [string]$response = Invoke-WebRequest -Uri "https://$target_ip/a79.htm" -UseBasicParsing
    $lines = $response.Split("`n")
    $myip = ""
    foreach ($line in $lines) {
        if ($line.Contains("ss5")) {
            $par = $line.Split("=")
            foreach ($i in $par) {
                if ($i.Contains("ss6")) {
                    $myip = $i.Split('"')[1]
                }
            }
            break
        }
    }
    if (!$myip.Contains("10.")) {
        # 未获取到ip，尝试空ip登录
        $myip = ""
    }
    Write-Host Got your ip = $myip

    # 登录逻辑
    $login_uri = "https://$target_ip"+":802/eportal/portal/login?callback=dr1003&login_method=1&user_account=%2C0%2C${identity}%40${operator}&user_password=${password}&wlan_user_ip=${myip}&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac"
    Write-Host Trying to contact with $login_uri
    $response = Invoke-WebRequest -Uri $login_uri -Headers @{
        "accept" = "*/*"
        "cookie" = "program=2; vlan=0; ssid=null; areaID=null; ip=${myip}"
    }
    # 检查结果
    Write-Host $response.ToString()
}

login
```

### 注解
如果上面的ip获取不正常，你可以尝试空ip，或者手动抠出来
你可以先用 `ifconfig {你的网卡} | awk -F ' *|:' '/inet addr/{print $4}'`检查输出的ip是否是你需要的.\
另外，对于部分Linux，`ifconfig eth0 | awk -F ' *|:' '/inet /{print $3}'`就可以了\
附： 例如你的ifconfig输出了这样的结果：
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.28.35.52  netmask 255.255.240.0  broadcast 172.28.47.255
        inet6 fe80::215:5dff:feb7:aca0  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:b7:ac:a0  txqueuelen 1000  (Ethernet)
        RX packets 323  bytes 233020 (233.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 122  bytes 8612 (8.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
eth0就是你的网卡名称，awk后面由 / / 包住的部分就应该是inet，就是IP地址前面的那些内容，很明显这里是第三个部分所以是$3，当然你也可以直接试出来

### 一个小问题
如果一直报密码错而且您的密码里有特殊字符的话（尤其是 `+` ），请手动将 `'${password}'` 替换为 URL Encode 后的密码字符串（可以直接抓包手动登陆时的URL）。

## 2. 发送一条短信验证码
### Code
校园网登录系统的任何页面，在调试控制台中输入并运行
```javascript
var url = "https://p.njupt.edu.cn:802/eportal/portal/sms" // 获取验证码链接
var phone = "" // 手机号
var data = {
    'telephone': phone,
    'mac': term.mac,
    'ip': term.ip,
    'ipv6': term.ipv6,
    'bind': 0,
    'page_index': page.index,
    'prefix': '',
    'sms_type': 0,
}
util._jsonp({
    url: url,
    data: data,
    success: function (json) {
      if (json.result == 1 || json.result == 'ok') { // result 1 发送成功 0 发送失败
        _alert(lang('验证码正下发，请注意查收！'));
      } else {
        _alert(lang(json.msg) || lang('发送验证码失败，因为网络问题或者页面已过期，请刷新页面重试！<br/>如果已收到验证码，请忽略此提示！'));
      }
    },
    error: function (error) {
      _alert(lang('发送验证码失败，因为网络问题或者页面已过期，请刷新页面重试！<br/>如果已收到验证码，请忽略此提示！'));
    }
  });
```
结果会出现在一个弹窗中\
冷知识：这个代码直接从相关js里面扣下来的，util类都是直接用的现成的

## 3. 获取用户在线信息
|变量名称|解释|例子|
|-------|---|----|
|target_ip|目标ip，10进制格式|175256324|

例如这里是我随便找的转换网站：https://www.browserling.com/tools/ip-to-dec
### Code
```bash
target_ip = ''
curl "https://p.njupt.edu.cn:802/eportal/portal/online_list?callback=dr1002&user_account=&user_password=&wlan_user_mac=000000000000&wlan_user_ip='${target_ip}'&curr_user_ip=&jsVersion=4.X&v=10081&lang=zh"
```
callback有两种\
dr1002({"result":0,"msg":"获取用户在线信息数据为空！"});\
dr1002({"result":1,"msg":"获取用户在线信息成功！","list":[{"online_session":某种idx,"online_time":"上线时间","online_ip":"ip","online_mac":"mac地址","time_long":"上线时长","uplink_bytes":"字面意思","downlink_bytes":"字面意思","dhcp_host":"","device_alias":"","nas_ip":"不知道啥意思，是个数字ip","user_account":"用户名@运营商","is_owner_ip":"固定分配此IP？"}],"total":1});

## 4. 登出用户
|变量名称|解释|例子|
|-------|----|---|
|target_ip|目标ip|10.114.51.4|

### Code
```bash
curl "https://p.njupt.edu.cn:802/eportal/portal/logout?callback=dr1003&login_method=1&user_account=drcom&user_password=123&ac_logout=1&register_mode=1&wlan_user_ip='${target_ip}'&wlan_user_ipv6=&wlan_vlan_id=0&wlan_user_mac=000000000000&wlan_ac_ip=&wlan_ac_name=&jsVersion=4.1.3&v=6215&lang=zh"
```

callback:\
dr1003({"result":0,"msg":"Radius注销失败！"});\
dr1003({"result":1,"msg":"Radius注销成功！"});