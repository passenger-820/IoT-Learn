参考视频 [ESP8266的Web配网教程【程序详解＋演示】](https://www.bilibili.com/video/BV187411H7et)



1. 软件平台：Arduino IDE
2. 硬件平台：ESP8266（NodeMCU）
3. 功能：web配网，和自动连接
4. 工作流程：开机后自动连接配置的WiFi（LED周期1s闪烁），如果不范围内，或连接超时； 则进入web配网模式（LED常亮）；连接成功（LED周期200ms闪烁）

> HTML

HTML代码压缩网址：https://www.sojson.com/jshtml.html

注意**一些符号需要添加转义符**

网页里填的是希望让开发板连上的WiFi（通常就是联网的WiFi），不是开发板自己的WiFi，开发板自己的WiFi是为了让别人连上他，帮助他去连网。毕竟开发板没键盘啊，不会自己敲字啊，得让别人帮自己一把。

```html
<!DOCTYPE html>
<html>
	<head>
		<!--<meta http-equiv="Content-Type" content="text/html; charset=GBK" />-->
		<meta charset="GBK">
		<meta name="viewport"content="width=device-width, initial-scale=1.0">
		<meta http-equiv="X-UA-Compatible"content="ie=edge">
		<title>不知名up的ESP8266网页配网</title>
		
	</head>
	<body>
		<form name="my">
			WiFi名称：
			<input type="text" name="s" placeholder="请输入您WiFi的名称" id="aa">
			 <br>
			WiFi密码：
			<input type="text" name="p" placeholder="请输入您WiFi的密码" id="bb">
			 <br>
			<input type="button" value="连接" onclick="wifi()">
		</form>
		<script language="javascript">
			
			function wifi()
			{
				var ssid = my.s.value;
				var password = bb.value;
				var xmlhttp=new XMLHttpRequest();
				xmlhttp.open("GET","/HandleVal?ssid="+ssid+"&password="+password,true);
				xmlhttp.send()	
				alert("用户名为："+ssid+' '+"密码为："+password);
			}
		</script>
	</body>
</html>
```

> 开发板

```c++
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <ESP8266mDNS.h>

#ifndef STASSID //这是用来让别人连他的，它通过这个来开启自己，别人用这个与他建立wifi通信
#define STASSID "qwer"
#define STAPSK  "asdfghjkl"
#endif

const char* ssid = STASSID;
const char* password = STAPSK;

ESP8266WebServer server(80);
bool LED_Flag = false;
String str = 
"<!DOCTYPE html><html><head><meta charset=\"UTF-8\"><meta name=\"viewport\"content=\"width=device-width, initial-scale=1.0\"><meta http-equiv=\"X-UA-Compatible\"content=\"ie=edge\"><title>不知名up的ESP8266网页配网</title></head><body><form name=\"my\">WiFi名称：<input type=\"text\"name=\"s\"placeholder=\"请输入您WiFi的名称\"id=\"aa\"><br>WiFi密码：<input type=\"text\"name=\"p\"placeholder=\"请输入您WiFi的密码\"id=\"bb\"><br><input type=\"button\"value=\"连接\"onclick=\"wifi()\"></form><script language=\"javascript\">function wifi(){var ssid=my.s.value;var password=bb.value;var xmlhttp=new XMLHttpRequest();xmlhttp.open(\"GET\",\"/HandleVal?ssid=\"+ssid+\"&password=\"+password,true);xmlhttp.send()}</script></body></html>";
/*****************************************************
 * 函数名称：handleRoot()
 * 函数说明：客户端请求回调函数
 * 参数说明：无
******************************************************/
void handleRoot() {
  server.send(200, "text/html", str);
}
/*****************************************************
 * 函数名称：HandleVal()
 * 函数说明：对客户端请求返回值处理，就是处理JS发送过来的信息
 * 参数说明：无
******************************************************/
void HandleVal()
{
    String wifis = server.arg("ssid"); //从JavaScript发送的数据中找ssid的值
    String wifip = server.arg("password"); //从JavaScript发送的数据中找password的值
    Serial.println(wifis); Serial.println(wifip);
    WiFi.begin(wifis,wifip);
}
/*****************************************************
 * 函数名称：handleNotFound()
 * 函数说明：响应失败函数
 * 参数说明：无
******************************************************/
void handleNotFound() {
  digitalWrite(LED_BUILTIN, 0);
  String message = "File Not Found\n\n";
  message += "URI: ";
  message += server.uri();
  message += "\nMethod: ";
  message += (server.method() == HTTP_GET) ? "GET" : "POST";
  message += "\nArguments: ";
  message += server.args();
  message += "\n";
  for (uint8_t i = 0; i < server.args(); i++) {
    message += " " + server.argName(i) + ": " + server.arg(i) + "\n";
  }
  server.send(404, "text/plain", message);
  digitalWrite(LED_BUILTIN, 1);
}
/*****************************************************
 * 函数名称：autoConfig()
 * 函数说明：自动连接WiFi函数，即自动连接存在8266里面的所有WiFi
 * 参数说明：无
 * 返回值说明:true：连接成功 false：连接失败
******************************************************/
bool autoConfig()
{
  WiFi.mode(WIFI_STA); // 先把WiFi模式设置为STA
  WiFi.begin(); // 自动连接寸的wifi
  Serial.print("AutoConfig Waiting......");
  for (int i = 0; i < 20; i++)  // 超时就会最后返回false，咱可以不用这么长的延迟时间
  {
    if (WiFi.status() == WL_CONNECTED)
    {
      Serial.println("AutoConfig Success");
      Serial.printf("SSID:%s\r\n", WiFi.SSID().c_str());
      Serial.printf("PSW:%s\r\n", WiFi.psk().c_str());
      WiFi.printDiag(Serial);
      return true;
      //break;
    }
    else
    {
      Serial.print(".");
      LED_Flag = !LED_Flag;
      if(LED_Flag)
          digitalWrite(LED_BUILTIN, HIGH);
      else
          digitalWrite(LED_BUILTIN, LOW); 
      delay(500);
    }
  }
  Serial.println("AutoConfig Faild!" );
  return false;
  //WiFi.printDiag(Serial);
}
/*****************************************************
 * 函数名称：htmlConfig()
 * 函数说明：web配置WiFi函数
 * 参数说明：无
******************************************************/
void htmlConfig()
{
    WiFi.mode(WIFI_AP_STA);//设置模式为AP+STA，即别人可以连他，他也可以连别人
    digitalWrite(LED_BUILTIN, LOW);
    WiFi.softAP(ssid, password); // 因为把他设置成了路由嘛，需要向别人提供自己的账号和密码，以方便别人通过此内容连接他
    Serial.println("AP设置完成");
    
    IPAddress myIP = WiFi.softAPIP();
    Serial.print("AP IP address: ");
    Serial.println(myIP);
  
    if (MDNS.begin("esp8266")) {
      Serial.println("MDNS responder started");
    }
  
    // 接着就是服务器响应
    server.on("/", handleRoot); // 访问192.xxx.xxx.xxx/ , 然后执行上面的handleRoot函数
    server.on("/HandleVal", HTTP_GET, HandleVal); // GET访问192.xxx.xxx.xxx/HandleVal,这就是HTML中的JS调用的wifi()方法, 然后执行上面的HandleVal函数
    server.onNotFound(handleNotFound);//请求失败回调函数
  
    server.begin();//开启服务器
    Serial.println("HTTP server started");
    while(1)
    {
        server.handleClient();
        MDNS.update();  
        if (WiFi.status() == WL_CONNECTED)
        {
            Serial.println("HtmlConfig Success");
            Serial.printf("SSID:%s\r\n", WiFi.SSID().c_str());
            Serial.printf("PSW:%s\r\n", WiFi.psk().c_str());
            Serial.println("HTML连接成功");
            break;
        }
    }  
}

void setup(void) {
    pinMode(LED_BUILTIN, OUTPUT);
    digitalWrite(LED_BUILTIN, HIGH);
    Serial.begin(115200);
    
    bool wifiConfig = autoConfig();
    // 自动配网失败，就执行web配网
    if(wifiConfig == false)
        htmlConfig();//HTML配网
}

void loop(void) {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(100);
    digitalWrite(LED_BUILTIN, LOW);
    delay(100);
}
```

