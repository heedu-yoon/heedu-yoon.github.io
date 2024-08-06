---
layout: post
title: WebSocket 요청 전송하기(javascript)
date: 2024-01-02 19:20:23 +0900
category: client
---

WebSocket을 사용하여 요청 전송이 필요한 경우가 있다.

간단하게 브라우저에서 지원하는 websocket을 이용해 보았다.

webSocket은 한번 연결이 되면 close를 할때 까지 여러번 요청을 주고받을 수 있는 점이 HTTP요청을 사용할 때와의 차이점인데 나는 요구사항이 있었기 때문에 HTTP요청과 비슷하게 동작하도록 한번 응답을 받으면 socket을 종료하도록 하였다.

```javascript
websocketModule = (function () {
var url;
var connTimeout = 30 * 1000;
var readTimeout = 30 * 1000;
var requestObj = {};

    var setConnectTimeout = function (timeout) {
        connTimeout = timeout;
    };

    var setReadTimeout = function (timeout) {
        readTimeout = timeout;
    };

    var addParameter = function (key, obj) {
        requestObj[key] = obj;
    };

    var setRequestObj = function (obj) {
        requestObj = obj;
    };

    var getRequestObj = function () {
        return requestObj;
    };

    var invoke = function (callback) {

        if (requestObj == undefined) {
            callback({ data: 'request is null' });
            return;
        }

        var socket = new WebSocket(url);

        var connectTimeout = setTimeout(function () {
            callback({ data: 'Socket connection timeout.' });
            socket.close();
        }, connTimeout);

        var readtimeout = setTimeout(function () {
            callback({ data: 'Socket read timeout.' });
            socket.close();
        }, readTimeout);

        socket.onopen = function (e) {
            clearTimeout(connectTimeout);
            socket.send(JSON.stringify(requestObj));
        };

        socket.onmessage = function (e) {
            clearTimeout(readtimeout);
            callback(e);
            socket.close();
        };

        socket.onerror = function (e) {
            clearTimeout(connectTimeout);
            clearTimeout(readtimeout);
            callback(e);
            socket.close();
        };
    };

    var Constructor;

    Constructor = function (url) {
        this.url = url;
    };

    Constructor.prototype = {
        invoke: function (callback) {
            invoke(callback);
        },
        addParameter: function (key, obj) {
            addParameter(key, obj);
        },
        setRequestObj: function (obj) {
            setRequestObj(obj);
        },
        getRequestObj: function () {
            return getRequestObj();
        },
        setConnectTimeout: function (timeout) {
            setConnectTimeout(timeout);
        },
        setReadTimeout: function (timeout) {
            setReadTimeout(timeout);
        }
    };

    return Constructor;
}());
```