前端代码：
```js
$(document).ready(function() {
	var ws = new WebSocket("ws://127.0.0.1:8080/echo");
	ws.onopen = function (evt) {
		var response = evt.data;
		log("收到消息：", response);
	};
	ws.onclose = function() {
		log("连接已关闭");
	};
	ws.onerror = function(err) {
		log("发生错误：", err);
	};
	$("#send").click(function() {
		var content = $('#content').val();
		log("正在发送数据", content);
		ws.send(content);
	});
});
```

```cpp
inline std::string websocketSecretHash(std::string userKey) {
	SHA1 sha1;
	std::string inKey = userKey + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
	sha1.add(inKey.data(), inKey.size());
	unsigned char buf[SHA1::HashBytes];
	sha1.getHash(buf);
	return base64::encode_into<std::string>(buf, buf + SHA1::HashBytes);
}
```