# 0429 Facebook 小遊戲
## 簡介
### WebSocket 介紹
### WebSocket 範例
## 第一部分－配對
第一部分我們先來做配對的部分，我們給第一位玩家專用的 URL ，等到第二位玩家存取這個網址後，檢查好友關係，並送出訊息讓網頁導向遊戲畫面。
### 流程圖
為了方便，我們在每次的通訊都會加上 **type** 這個值，用來說明這次的通訊目的，下圖中左、右方的文字就是 type 。

![](https://nodec4.azurewebsites.net/pic/1.svg)
- 第一位玩家必須由**1.建立遊戲**來取得遊戲ID，並讓自己加入。
- 第二位玩家則需要提供遊戲ID來加入。
- 兩個人都加入後，伺服器會發送 **MATCH** 訊息給所有人。

### 登入流程
Facebook 的登入流程相信大家已經很熟悉了，這裡就不多做解釋，程式部分如下。
```javascript
app.get('/api/code', (req, res) => {
    request('https://graph.facebook.com/v2.8/oauth/access_token?client_id=' + process.env.appID + '&redirect_uri=' + process.env.redirect + '/api/code?id=' + req.query.id + '&client_secret=' + process.env.appKEY + '&code=' + req.query.code, (error, response, body) => {
        
        var userdata = JSON.parse(body);
        req.session.key = userdata.access_token;
        /*  */
        getUser(userdata.access_token).then((data) => {
            req.session.name = data.name;
            req.session.fbid = data.id;
			// #27
            var url = (req.query.id != 'undefined')? '../?id=' + req.query.id : '../';
            res.redirect(url);
        });
    });
});
function getUser(key) {
    return new Promise((resolve, reject) => {
        request('https://graph.facebook.com/v2.8/me?fields=id%2Cname&access_token=' + key, (error, response, body) => {
            resolve(JSON.parse(body));
        });
    });
}
```
> 在 27 行我們判斷網址中是不是有 id 這個值，若有的話我們得一起傳到下個網頁去，否則使用者登入之後會遺失遊戲 id 。
> [color=blue]

### WebSocket 中如何使用 Session
還記得接收到 Code 之後我們做的事情嗎? 我們把使用者的 id,name,access_token 存到 Session 中方便後續使用，這時候就會有個小問題出現囉，WebSocket (下方簡稱WS) 收到訊息後，是沒有辦法直接存取到 Session 的。但我們可以使用 SessionParser 取得資料內容後傳給 WS，方法有很多種，程式碼如下
```javascript
const wss = new WebSocket.Server({ server, path: "/ws" });
wss.on('connection', function connection(ws) {
    sessionParser(ws.upgradeReq, {}, function () {
        var session = ws.upgradeReq.session;
        ws.session = session;
    });
    ws.on('message', function incoming(data) {
       // 處理接收到的訊息
    });
});
```
我們在 WS 連接 (on connection) 時，存取 session ，並把讀取到的資料傳給 ws.session 中。
> 我們是在 WS 連接時存取資料，所以之後若 session 有更新，已連線的 WS 是沒有辦法獲得新資料的，但在這個範例我們不在乎這點，因為我們要的資料只有 Facebook 資訊，這些資料早在 WS 連接前就寫入了。
> [color=red]

### Module
在前幾個範例中，我們把所有的程式都寫在 ==app.js== 中，這不是件好事情，這次我們分成 4 個檔案。
|檔案名稱|說明|
|----|----|
|app.js|程式進入點，開啟伺服器，route設定|
|wsConnect.js|處理 WebSocket 通訊(接收和傳送)|
|gameManager.js|遊戲流程，建立遊戲房間，要求題目，評分|
|question.js|負責出題目|
這麼一來程式架構就比較清楚了，那不同檔案之間要怎麼溝通呢?
我們來看看下面這個例子，1.js 中呼叫 2.js 的 Function
**1.js**
```javascript
var secondJs = require('./2')
secondJs.hello();
```
**2.js**
```javascript
function printHelloWord(){
	console.log("Hello Word");
}
module.exports = {
	hello: printHelloWord
}
```
4 ~ 6 行中，我們定義了 2.js 的輸出，這麼一來，在 1.js 中呼叫  secondJs.hell 就會對應到 2.js 的 printHelloWord 了。
在接下來的程式中，為了方便，我們會將輸出的名稱和真正的名稱設為相同，像這樣
```javascript
module.exports = {
	printHelloWord: printHelloWord
}
// 名稱相同時，我們可以簡寫成下面這個樣子
module.exports = {
	printHelloWord
}
```

### 後端程式
#### 建立 WebSocket 伺服器
在這個範例中，我們會使用 Express 傳送靜態檔案，使用 WebSocket 來進行資料傳輸，這邊主要講解 WS 的部分。
```javascript
//#41
const server = http.createServer(app);
//#42
const wss = new WebSocket.Server({ server, path: "/ws" });
wss.on('connection', function connection(ws) {
    sessionParser(ws.upgradeReq, {}, function () {
        var session = ws.upgradeReq.session;
        ws.session = session;
    });
	//#48
    ws.on('message', function incoming(data) {
        var msg = { type: null };
        try {
            msg = JSON.parse(data);
        } catch (e) { }
        if (msg.type == null) {
            ws.close();
        }
        else {
            wsc.route(ws, msg);
        }

    });
});
```
41 行 createServer(app)：載入 Express 的設定
42 行 WebSocket.Server：在路徑 /ws 下建立 WS 伺服器，我們使用 [ws](https://github.com/websockets/ws) 模組。
```javascript
var WebSocket = require('ws');
```
48 ws.on('message'...)：行定義了接收到訊息的動作，前面有提到為了方便，我們每次通訊都會加上 type 屬性，在接收到訊息後我們先檢查這個屬性，關閉不相容的通訊，接把訊息交給 **wsConnection.js** 處理，當然我們也先引用我們自己的模組。
```javascript
var wsc = require('./wsConnect');
```
> 若您使用 Azure Web Service 作為平台，要先到設定面板中開啟 Websocket 功能才可以使用。
> 
> Azure Web Service 會限制 WebSocket 的連線數量，詳細資訊可以在[這裡](https://docs.microsoft.com/en-us/azure/azure-subscription-service-limits)找到，免費的方案只有 5 個連線數，所以關閉不使用的連線是個聰明的選擇。
> [color=blue]

#### 下載檔案
前面有提到，關於遊戲管理的部分，都由 ==gameManager.js== 處理，我們沒有辦法在這裡教學這部分，麻煩讀者到 [GitHub](https://raw.githubusercontent.com/oscar60310/NodeJsClass4/master/gameManager.js) 下載這個檔案，並放在和 app.js 同一個目錄的位置。

#### 處理通訊
首先，我們得先引用 gameManager ，並定義好 route 函式(用來處理接收到的訊息)以及輸出，另外我們 WS 訊息會以 json 格式傳送，為了方便，我們先寫好傳送 JSON 的功能。
```javascript
const gm = require('./gameManager');
route = (ws, msg) => {
    switch (msg.type) {
		//依照不同類別處理訊息
	}
}
// 傳送JSON格式資料
sendJson = (ws, msg) => {
    ws.send(JSON.stringify(msg));
}
// 定義輸出
module.exports = {
    route
}
```
在配對的頁面中，我們會接到三種訊息，分別是 **LOGIN** 檢查登入狀態、**CREATE** 建立遊戲房間以及 **JOIN** 加入遊戲。

檢查登入狀態和以前相通，有登入就回傳名子，沒有就給授權網址，指示改為 WS 發送而已。但在回呼網址中，我們加上了 id 參數，這是為了讓有遊戲 ID 但還未登入的使用者保留 ID，而不會在登入後(網頁重新導向)遺失這個參數。
```javascript
case "LOGIN":
if (ws.session.name)
	sendJson(ws, { type: msg.type, status: 'ok', name: ws.session.name });
else
	sendJson(ws, { type: msg.type, status: 'not login', url: 'https://www.facebook.com/v2.8/dialog/oauth?client_id=' + process.env.appID + '&redirect_uri=' + process.env.redirect + '/api/code?id=' + msg.game + '&scope=user_friends,user_birthday,user_photos' });
break;
```
接著是建立遊戲 ID 的部分了，用法很簡單，呼叫 createNewGame 即可
```javascript
case "CREATE":
if (ws.session.name)
	sendJson(ws, { type: msg.type, status: 'ok', id: gm.createNewGame(ws), name: ws.session.name });
else
	sendJson(ws, { type: msg.type, status: "not login" });
break;
```
> 建立的遊戲 ID 在十分鐘後會自動刪除
> [color=pink]

最後是加入遊戲的部分了，值得注意的是兩個人(不管是不是建立遊戲的人)都需要執行才能開始遊戲。
用法為 joinGame([Websocket],[Game ID]) ，為 promise，會回傳 status (是否成功加入) 及 todo，若 todo 為 ==notify== 時，表示已經有兩個人加入，可以開始遊戲，這時候我們必須傳送 MATCH 指令到前端。
```javascript
case "JOIN":
if (msg.game && ws.session.name) {
	gm.joinGame(ws, msg.game).then((d) => {
		ws.session.game = msg.game;
		sendJson(ws, { type: msg.type, status: d.status, id: msg.game });
		if (d.todo == 'notify') {
			d.notify.forEach((wsToNotify) => {
			sendJson(wsToNotify, { type: "MATCH", players: d.players });
			})
		}
	});
} else {
	sendJson(ws, { type: msg.type, status: "not login or no game id" });
}
break;
```

### 前端程式
因為這次很多和前端的互動，我們得知道前後端怎麼運作的，所以這次我們會講解一些前端的東西。
#### 開啟 WebSocket
前端開啟 ws 很簡單 
```javascript
new WebSocket("ws://[url]");
```
但考慮到我們可能在 HTTPS 連線下使用(這時須使用 wss://)，所以我們把程式改寫成這樣
```javascript
var protocol = (window.location.protocol == "https:") ? "wss://" : "ws://";
ws = new WebSocket(protocol + window.location.host + "/ws");
```
後面的 /ws 是我們在伺服器設定的路徑
接著我們定義 onopen(開啟連線)、onmessage(收到訊息)、onclose(關閉連線)的動作
```javascript
self.ws.onopen = function () {
        // Web Socket is connected, send data using send()
        console.log("[ws] connected.");
        self.sendJson(self.ws, {
            type: "LOGIN"
        });
};
self.ws.onmessage = function (evt) {
    var msg = JSON.parse(evt.data);
    console.log(msg);
    switch (msg.type) {
       case "LOGIN":
            loginStatus(msg);
            break;
        case "CREATE":
            newgame(msg);
            break;
        case "JOIN":
            checkJoin(msg);
            break;
        case "MATCH":
            match(msg);
            break;
    }

};
this.ws.onclose = function () {
    console.log("ws closed.");
};
this.sendJson = (ws, msg) => {
    ws.send(JSON.stringify(msg));
}
```
首先，我們一樣定義了 sendJson 來傳送資料，我們把所有的內容包在 **WsManager** 中，方便我們使用，並且在最上方定義了 self = WsManager，這只是我們的習慣，讀者可以自行決定。

建立完成連線後，我們馬上檢查登入，並等待伺服器回傳資訊。登入之後，我們先試試看能不能利用 ID 加入遊戲
```javascript
loginStatus = (msg, text, url) => {
        if (msg.status == "ok") {
            self.sendJson(self.ws, {
                type: "JOIN",
                game: getUrlParameter("id")
            });
        } else {
            setAlert("<a href='" + msg.url + "' class=\"btn btn-primary btn-lg\" role=button>玩家FB登入</a>");
        }
}
```
若沒辦法加入，我們自己建立一個
```javascript
checkJoin = (msg) => {
        if (msg.status != "ok") {
            setAlert("無法加入", "ID 無效，產生新的房間...", "");
            setTimeout(() => self.sendJson(self.ws, {
                type: "CREATE"
            }), 1000);
        } else
            self.gid = msg.id;
    }
```
加入完成後，就等待伺服器回傳 MATCH 指令就可以了！
這部分的完整程式在[這裡](https://github.com/oscar60310/NodeJsClass4/blob/master/static/js/Match_web.js)

### 小節
到這裡我們已經完成了配對，接著就遊戲的內容了，配對完成後，前端要把兩位使用者導向 room.html 這個網頁，並附上遊戲 ID
![](https://i.imgur.com/Rch44n1.png)
原諒我們沒有時間美化介面^^
> 配對的流程需要兩個人(廢話)，若您只有一個人...可以開兩個瀏覽器或兩頁分頁
> [color=pink]

## 第二部分－遊戲
### 流程圖
![](https://nodec4.azurewebsites.net/pic/2.svg)
1. 使用者加入後會傳送 INFO 要求取得遊戲資訊(兩位玩家的名稱和照片)
2. 當兩位玩家都傳送要求後，伺服器會開始製作題目，製作完成時發送 COMPUTING (finish: true)
3. 接著前端必須發送 READY 要求(當使用者按下**準備**)
4. 伺服器會發送 START 訊息，五秒後會發送題目
5. 伺服器發送題目，前端回傳答案，伺服器公布結果，重複執行值到沒有題目
6. 結束遊戲

### 取得題目
#### INFO
取得遊戲資訊，使用方式很為 info([WebSocket],[Game ID])
回傳 data (遊戲資訊內容)、todo (當todo為notify時，表示有兩位玩家要求 info ，開始準備題目)
```javascript
case "INFO":
if (ws.session.name && msg.game) {
	gm.info(ws, msg.game).then((d) => {
		d.data.type = msg.type;
		ws.gameid = msg.game;
		sendJson(ws, d.data);
		if (d.todo == "notify") {
			sendJson(d.ws[0], { type: "COMPUTING" });
     			sendJson(d.ws[1], { type: "COMPUTING" });
			gm.createQuestions(msg.game).then(notifyGetReady);
		}
	});
}
break;
```
題目開始準備和準備完成我們都會發送訊息給前端
```javascript
notifyGetReady = (data) => {
    var s = (data.status == "ok") ? { type: "COMPUTING", finish: true } : { type: "COMPUTING", finish: false };
    sendJson(data.game.players[0].ws, s);
    sendJson(data.game.players[1].ws, s);
}
```
#### 題目
要取得題目，只需要呼叫 getQuestion([Gamd ID]) 即可，會回傳 que (題目內容) 或 null (沒有剩下的題目了)，傳送題目的程式如下，若沒有剩下題目，我們則傳送 **END** 訊息到前端。
```javascript
sendQuestion = (gameid) => {
    gm.getQuestion(gameid).then((data) => {
        if (data != null) {
            var s = {
                type: "QUESTION",
                time: data.que.time,
                que: data.que.description,
                ans: data.que.ans,
                id: data.que.id
            };
            sendJson(data.d[0], s);
            sendJson(data.d[1], s);
            var g = gm.getGameByID(gameid);
            setTimeout(sendResult, s.time * 1000, g, g.nowquestion);
        }
        else {
            var game = gm.getGameByID(gameid);
            var re = { type: "END", data: gm.getEndResult(game) };
            for (var i = 0; i < 2; i++) {
                re.id = i;
                sendJson(game.players[i].ws, re);
            }

        }

    });
}
```
我們在送完題目後馬上開啟計時器(setTimeout)，這是為了在使用者超過時間還沒有回傳答案時，公布結果。


### 計分
#### 回答問題
回答問題時，使用 solveQuestion([WS],[Answer])
程式會判斷是哪位使用者回答(相同Facebook ID可能會出錯)
使用者回答後，我們隔 1 秒公布答案。
```javascript
 gm.solveQuestion(ws, msg.choose).then((data) => {
	if (data) {
		var g = gm.getGameByID(ws.gameid);
        	setTimeout(sendResult, 1000, g, g.nowquestion);
	}
}).catch((e) => { });
```
#### 公布答案
getResult([Game]) 可取得答案及計分，我們在程式中判斷 **nq** 是否等於 nowquestion ，這樣可以避免回答到其他題目的答案。
送出結果後，我們隔 2 秒，進行下一題。
```javascript
sendResult = (game, nq) => {
    if (game.nowquestion == nq) {
        var re = { type: "RESULT", data: gm.getResult(game) };
        for (var i = 0; i < 2; i++) {
            re.id = i;
            sendJson(game.players[i].ws, re);
        }
        setTimeout(sendQuestion, 2000, game.id);
    }
}
```


## 總結
![](https://i.imgur.com/MzHGB9B.png)


目前題目只會有兩種(猜生日和按讚)，讀者可以自行修改 question.js 中的程式來增加不同的題目！

謝謝您把文章看完，我們的系列課程也到一段落了，希望你們會喜歡。有任何其他問題，歡迎聯絡我們。
