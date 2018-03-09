---
title: webrtc通信过程
date: 2018-03-06 09:53:22
tags: 
      - webrtc
      - 前端
      - node.js
---

### WebRTC Api 

##### PeerConnection

WebRTC中最主要的就是一个叫做`PeerConnection`的对象，这个是WebRTC中已经封装好的对象。每一路的音视频会话都会有唯一的一个`PeerConnection`对象，WebRTC通过这个`PeerConnection`对象进行视频的发起、传输、接收和挂断等操作。
PeerConnection中包含的属性如下：

- localDescription：本地描述信息，类型：`RTCSessionDescription`
- remoteDescription：远端描述信息，类型：`RTCSessionDescription`
- onicecandidate：传入一个回调方法，该回调方法有一个返回参数，返回参数类型为：`RTCIceCandidateEvent`，
- onaddstream：传入一个回调方法，该回调方法有一个返回参数，返回参数类型为：``，如果检测到有远程媒体流传输到本地之后便会调用该方法。
- ondatachannel：（暂未用到）
- oniceconnectionstatechange：（暂未用到）
- onnegotiationneeded：（暂未用到）
- onremovestream：（暂未用到）
- onsignalingstatechange：（暂未用到）

`PeerConnection`中还包含了一些方法：

- setLocalDescription：设置本地offer，将自己的描述信息加入到`PeerConnection`中，参数类型：`RTCSessionDescription`
- setRemoteDescription：设置远端的`answer`，将对方的描述信息加入到`PeerConnection`中，参数类型：`RTCSessionDescription`
- createOffer：创建一个offer，需要传入两个参数，第一个参数是创建offer成功的回调方法，会返回创建好的offer，可以在这里将这个offer发送出去。第二个参数是创建失败的回调方法，会返回错误信息。
- createAnswer：创建一个`answer`，需要传入两个参数，第一个参数是创建`answer`成功的回调方法，会返回创建好的`answer`，可以在这里将这个`answer`发送出去。第二个参数是创建失败的回调方法，会返回错误信息。
- addIceCandidate：将打洞服务器加入到配置信息中，参数类型：`RTCIceCandidate`
- addStream：向`PeerConnection`中加入需要发送的数据流，参数类型：`MediaStream`
- close：
- createDTMFSender：
- createDataChannel：
- getLocalStreams：
- getRemoteStreams：
- getStats：
- getStreamById：
- removeStream：
- updateIce：

##### RTCSessionDescription

`RTCSessionDescription`类型中包含了两个属性：

- sdp：a read-only [`DOMString`](https://developer.mozilla.org/en-US/docs/Web/API/DOMString) containing the [SDP](https://developer.mozilla.org/en-US/docs/Glossary/SDP) which describes the session.

  ​	

- type：这个指明了是视频的接收方还是发起方

  ​	This enum defines strings that describe the current state of the session description, as used in the [`type`](https://developer.mozilla.org/en-US/docs/Web/API/RTCSessionDescription/type) property. The session description's type will be specified using one of these values.

  | Value      | Description                              |
  | ---------- | ---------------------------------------- |
  | `answer`   | The SDP contained in the [`sdp`](https://developer.mozilla.org/en-US/docs/Web/API/RTCSessionDescription/sdp) property is the definitive choice in the exchange. In other words, this session description describes the agreed-upon configuration, and is being sent to finalize negotiation. |
  | `offer`    | The session description object describes the initial proposal in an offer/answer exchange. The session negotiation process begins with an offer being sent from the caller to the callee. |
  | `pranswer` | The session description object describe a provisional answer; that is, it's a response to a previous offer or provisional answer. |
  | `rollback` | This special type with an empty session description is used to roll back to the previous stable state. |

### Peer to Peer Connection 通信过程

用户A合用户B建立连接的过程（websocket通信）

1. 新建websocket的连接

   ```javascript
   var conn = new WebSocket('ws://localhost:9090');
   ```

2. 创建iceserver

   ```javascript
   var configuration = {
                   "iceServers": [{ "urls": "stun:stun2.1.google.com:19302" }]
               };
   ```

3. A获取UserMedia

   A创建PeerConnection

   向本地视频中加入当前视频流stream

   向PeerConnection中加入该stream

   ```javascript
         navigator.getUserMedia = (
               navigator.getUserMedia ||
               navigator.webkitGetUserMedia ||
               navigator.mozGetUserMedia ||
               navigator.msGetUserMedia
           );

       navigator.getUserMedia({ audio: false, video: true }, function(stream) {

               //displaying local video stream on the page 
               localVideo.srcObject = stream;
     
   			// setup a new PeerConnection
               yourConn = new RTCPeerConnection(configuration);

               // setup stream listening 
               yourConn.addStream(stream);
           }, function(error) {
               console.log(error);
           });

   ```

4. A创建一个offer，把offer加入LocalDescription中

   ```javascript
   // create an offer 
           yourConn.createOffer(function(offer) {
               send({
                   type: "offer",
                   offer: offer
               });

               yourConn.setLocalDescription(offer);
           }
   ```

5. A监听iceCandicate服务器信息，收到event后把其中的candicate发送出去

   ```javascript
   // 发送ICE候选到其他客户端
               yourConn.onicecandidate = function(event) {
                   if (event.candidate) {
                       send({
                           type: "candidate",
                           candidate: event.candidate
                       });
                   }
               };
   ```

6. websocket的connection监听offer，

   如果B收到offer，

   B将offer存入peerConnection中setRemoteDescription, 并创建一个answer并返回给A：

   ```javascript
   //when we got a message from a signaling server 
   conn.onmessage = function(msg) {
       console.log("Got message", msg.data);

       var data = JSON.parse(msg.data);

       switch (data.type) {
           case "login":
               handleLogin(data.success);
               break;
               //when somebody wants to call us 
           case "offer":
               handleOffer(data.offer, data.name);
               break;
           case "answer":
               handleAnswer(data.answer);
               break;
               //when a remote peer sends an ice candidate to us 
           case "candidate":
               handleCandidate(data.candidate);
               break;
           case "leave":
               handleLeave();
               break;
           default:
               break;
       }
   };

   //when somebody sends us an offer 
   function handleOffer(offer, name) {
       connectedUser = name;
     	//offer存入peerConnection中,
       yourConn.setRemoteDescription(new RTCSessionDescription(offer));

       //create an answer to an offer 
       yourConn.createAnswer(function(answer) {
           yourConn.setLocalDescription(answer);

           send({
               type: "answer",
               answer: answer
           });

       }, function(error) {
           alert("Error when creating an answer");
       });
   };
   ```

7. B收到了A发来的iceCandicate，将IceCandidate加入连接中

   ```javascript
   //when we got an ice candidate from a remote user 
   function handleCandidate(candidate) {
       yourConn.addIceCandidate(new RTCIceCandidate(candidate));
   };
   ```

8. B开始获取视音频数据，将视音频数据存入`PeerConnection`中，WebRTC便会自动将视音频数据发送给A。

   ```javascript
   			//这里在连接建立的时候就提前设置好远程连接的监听，一旦B把A添加

               //when a remote user adds stream to the peer connection, we display it 
               yourConn.onaddstream = function(e) {
                   remoteVideo.srcObject = e.stream;
               };
   ```

9. A接收到B返回的`answer`，将B返回的`answer`设置为`PeerConnection`的`remoteDescription`。

   ```javascript
   //when we got an answer from a remote user
   function handleAnswer(answer) {
       yourConn.setRemoteDescription(new RTCSessionDescription(answer));
   };
   ```

10. 这个时候WebRTC会将视音频数据自动发送给B，A和B就建立起了实时视音频通信。







### 全部代码

```js
//our username 
var name;
var connectedUser;

//connecting to our signaling server
var conn = new WebSocket('ws://localhost:9090');

conn.onopen = function() {
    console.log("Connected to the signaling server");
};

//when we got a message from a signaling server 
conn.onmessage = function(msg) {
    console.log("Got message", msg.data);

    var data = JSON.parse(msg.data);

    switch (data.type) {
        case "login":
            handleLogin(data.success);
            break;
            //when somebody wants to call us 
        case "offer":
            handleOffer(data.offer, data.name);
            break;
        case "answer":
            handleAnswer(data.answer);
            break;
            //when a remote peer sends an ice candidate to us 
        case "candidate":
            handleCandidate(data.candidate);
            break;
        case "leave":
            handleLeave();
            break;
        default:
            break;
    }
};

conn.onerror = function(err) {
    console.log("Got error", err);
};

//alias for sending JSON encoded messages 
function send(message) {
    //attach the other peer username to our messages 
    if (connectedUser) {
        message.name = connectedUser;
    }

    conn.send(JSON.stringify(message));
};

//****** 
//UI selectors block 
//******

var loginPage = document.querySelector('#loginPage');
var usernameInput = document.querySelector('#usernameInput');
var loginBtn = document.querySelector('#loginBtn');

var callPage = document.querySelector('#callPage');
var callToUsernameInput = document.querySelector('#callToUsernameInput');
var callBtn = document.querySelector('#callBtn');

var hangUpBtn = document.querySelector('#hangUpBtn');

var localVideo = document.querySelector('#localVideo');
var remoteVideo = document.querySelector('#remoteVideo');

var yourConn;
var stream;

callPage.style.display = "none";

// Login when the user clicks the button 
loginBtn.addEventListener("click", function(event) {
    name = usernameInput.value;

    if (name.length > 0) {
        send({
            type: "login",
            name: name
        });
    }

});

function handleLogin(success) {
    if (success === false) {
        alert("Ooops...try a different username");
    } else {
        loginPage.style.display = "none";
        callPage.style.display = "block";

        //********************** 
        //Starting a peer connection 
        //********************** 

        //getting local video stream 
        navigator.getUserMedia = (
            navigator.getUserMedia ||
            navigator.webkitGetUserMedia ||
            navigator.mozGetUserMedia ||
            navigator.msGetUserMedia
        );

        navigator.getUserMedia({ audio: false, video: true }, function(myStream) {
            stream = myStream;

            //displaying local video stream on the page 
            localVideo.srcObject = stream;

            //using Google public stun server 
            var configuration = {
                "iceServers": [{ "urls": "stun:stun2.1.google.com:19302" }]
            };

            yourConn = new RTCPeerConnection(configuration);

            // setup stream listening 
            yourConn.addStream(stream);

            //when a remote user adds stream to the peer connection, we display it 
            yourConn.onaddstream = function(e) {
                remoteVideo.srcObject = e.stream;
            };

            // Setup ice handling ?????????
            yourConn.onicecandidate = function(event) {
                if (event.candidate) {
                    send({
                        type: "candidate",
                        candidate: event.candidate
                    });
                }
            };

        }, function(error) {
            console.log(error);
        });

    }
};

//initiating a call 
callBtn.addEventListener("click", function() {
    var callToUsername = callToUsernameInput.value;

    if (callToUsername.length > 0) {

        connectedUser = callToUsername;

        // create an offer 
        yourConn.createOffer(function(offer) {
            send({
                type: "offer",
                offer: offer
            });

            yourConn.setLocalDescription(offer);
        }, function(error) {
            alert("Error when creating an offer");
        });

    }
});

//when somebody sends us an offer 
function handleOffer(offer, name) {
    connectedUser = name;
    yourConn.setRemoteDescription(new RTCSessionDescription(offer));

    //create an answer to an offer 
    yourConn.createAnswer(function(answer) {
        yourConn.setLocalDescription(answer);

        send({
            type: "answer",
            answer: answer
        });

    }, function(error) {
        alert("Error when creating an answer");
    });
};

//when we got an answer from a remote user
function handleAnswer(answer) {
    yourConn.setRemoteDescription(new RTCSessionDescription(answer));
};

//when we got an ice candidate from a remote user 
function handleCandidate(candidate) {
    yourConn.addIceCandidate(new RTCIceCandidate(candidate));
};

//hang up 
hangUpBtn.addEventListener("click", function() {

    send({
        type: "leave"
    });

    handleLeave();
});

function handleLeave() {
    connectedUser = null;
    remoteVideo.src = null;

    yourConn.close();
    yourConn.onicecandidate = null;
    yourConn.onaddstream = null;
};
```



### websocket代码：

```js
//require our websocket library 
var WebSocketServer = require('ws').Server; 

//creating a websocket server at port 9090 
var wss = new WebSocketServer({port: 9090}); 

//all connected to the server users 
var users = {};
  
//when a user connects to our sever 
wss.on('connection', function(connection) {
  
   console.log("User connected");
	
   //when server gets a message from a connected user 
   connection.on('message', function(message) { 
	
      var data; 
		
      //accepting only JSON messages 
      try { 
         data = JSON.parse(message); 
      } catch (e) { 
         console.log("Invalid JSON"); 
         data = {}; 
      }
		
      //switching type of the user message 
      switch (data.type) { 
         //when a user tries to login
         case "login": 
            console.log("User logged", data.name); 
				
            //if anyone is logged in with this username then refuse 
            if(users[data.name]) { 
               sendTo(connection, { 
                  type: "login", 
                  success: false 
               }); 
            } else { 
               //save user connection on the server 
               users[data.name] = connection; 
               connection.name = data.name; 
					
               sendTo(connection, { 
                  type: "login", 
                  success: true 
               }); 
            } 
				
            break;
				
         case "offer": 
            //for ex. UserA wants to call UserB 
            console.log("Sending offer to: ", data.name);
				
            //if UserB exists then send him offer details 
            var conn = users[data.name]; 
				
            if(conn != null) { 
               //setting that UserA connected with UserB 
               connection.otherName = data.name; 
				//发给那个users[data.name]就是用户B，的conn，消息是用户A要给你发offer，里面带的name是用户A的名字	
               sendTo(conn, { 
                  type: "offer", 
                  offer: data.offer, 
                  name: connection.name 
               }); 
            }
				
            break;
				
         case "answer": 
            console.log("Sending answer to: ", data.name); 
            //for ex. UserB answers UserA 
            var conn = users[data.name]; 
			//这个conn的B消息里发的data.name也就是A的名字，发给a一个answer
            if(conn != null) { 
               connection.otherName = data.name; 
               sendTo(conn, { 
                  type: "answer", 
                  answer: data.answer 
               }); 
            } 
				
            break; 
				
         case "candidate": 
            console.log("Sending candidate to:",data.name);
            //同样，这是A发给B的candidate 
            var conn = users[data.name];
				
            if(conn != null) { 
               sendTo(conn, { 
                  type: "candidate", 
                  candidate: data.candidate 
               }); 
            } 
				
            break;
				
         case "leave": 
            console.log("Disconnecting from", data.name); 
            var conn = users[data.name]; 
            conn.otherName = null; 
				
            //notify the other user so he can disconnect his peer connection 
            if(conn != null) {
               sendTo(conn, { 
                  type: "leave" 
              }); 
            }
				
            break;
				
         default: 
            sendTo(connection, { 
               type: "error", 
               message: "Command not found: " + data.type 
            }); 
				
            break; 
      }
		
   }); 
	
   //when user exits, for example closes a browser window 
   //this may help if we are still in "offer","answer" or "candidate" state 
   connection.on("close", function() { 
	
      if(connection.name) { 
         delete users[connection.name]; 
			
         if(connection.otherName) { 
            console.log("Disconnecting from ", connection.otherName); 
            var conn = users[connection.otherName]; 
            conn.otherName = null;
				
            if(conn != null) { 
               sendTo(conn, { 
                  type: "leave" 
               }); 
            }
         } 
      }
		
   });  
	
   connection.send("Hello world");  
});
  
function sendTo(connection, message) { 
   connection.send(JSON.stringify(message)); 
}
```

