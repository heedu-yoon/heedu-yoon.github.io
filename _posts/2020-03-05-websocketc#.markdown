---
layout: post
title: WebSocket 수신부 만들기
date: 2024-03-05 19:20:23 +0900
category: c#
---

WebSocket수신부를 만들어야 할 일이 있어서 간략하게 구현 해 보았다.

약속된 구조대로 주고받으면 되기 때문에 tcp로 받아서 byte별로 읽어 처리하면 된다.

데이터 타입이 여러 형태가 있고 나는 text형태의 데이터만 수신하도록 구현하였다.

한가지 더 필요한 점은 Client에서 socket을 끊을때 close요청이 전달되게 되기 때문에 text와 close를 받을 수 있게 하였다.

```csharp
using System;
using System.Collections;
using System.Threading;
using System.IO;
using System.Net.Sockets;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;
using System.Text;
using System.Text.RegularExpressions;

namespace Crownix.AgentLib
{
// websocket data Frame format (https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)
/*
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
*/

    public enum PayloadDataType
    {
        Unknown = -1,
        Text = 1,
        ConnectionClose = 8
    }

    public class WebSocketHandler
    {
        protected TcpClient client = null;
        protected X509Certificate2 cert2 = null;

        public WebSocketHandler(TcpClient client, X509Certificate2 cert2 = null)
        {
            this.client = client;
            this.cert2 = cert2;
        }

        public override void Process()
        {
            Stream clientStream = null;

            bool isClosed = false;

            clientStream = getStream();

            if (clientStream == null)
                return;

            while (!isClosed)
            {
                try
                {

                    byte[] reqData = readMessage(clientStream);
                    string message = Encoding.UTF8.GetString(reqData);

                    PayloadDataType opcode = (PayloadDataType)(reqData[0] & 15); // 000011111

                    if (Regex.IsMatch(message, "^GET", RegexOptions.IgnoreCase)) // 'GET' 요청으로 Handshake
                    {
                        HandshakeToClient(clientStream, message);
                        continue;
                    }

                    if (opcode == PayloadDataType.Text) // 요청은 text만 수신
                    {
                        string requestString = requestProcess(reqData);
                        // process request
                    }
                    else if (opcode == PayloadDataType.ConnectionClose) // close요청이 오면 socket을 종료함.
                    {
                        isClosed = true;
                        break;
                    }
                    else
                        throw new Exception("Unknown Data Type");
                }
                finally
                {
                    if (isClosed)
                        clientStream.Dispose();
                }

                // write response
            }
        }

        private string requestProcess(byte[] reqData)
        {
            bool mask = (reqData[1] & 128) != 0; // 클라이언트는 mask bit(8번 bit)를 항상 1로 준다.(true)
            string result = null;

            int msglen = reqData[1] - 128; // 9~15 bit는 데이터 길이
            int offset = 2;

            if (msglen == 126) // 126인 경우 뒤에 2개 byte가 데이터 길이
            {
                msglen = BitConverter.ToUInt16(new byte[] { reqData[3], reqData[2] }, 0);
                offset = 4;
            }

            if (mask)
            {
                byte[] decoded = new byte[msglen];
                byte[] masks = new byte[4] { reqData[offset], reqData[offset + 1], reqData[offset + 2], reqData[offset + 3] };
                offset += 4;

                // mask 해제 공식 (Di = Ei XOR M(i mod 4))
                for (int i = 0; i < msglen; ++i)
                    decoded[i] = (byte)(reqData[offset + i] ^ masks[i % 4]);

                string text = Encoding.UTF8.GetString(decoded);
                result = text;
            }
            else // 클라이언트 -> 서버로 전달하는 요청에 경우 항상 mask가 true(https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_server)
                throw new Exception("Invalid mask bit.");

            return result;
        }

        private byte[] readTcpMessage(NetworkStream clientStream)
        {
            while (!clientStream.DataAvailable) ;

            byte[] bytes = new byte[client.Available];
            clientStream.Read(bytes, 0, client.Available);

            return bytes;
        }

        private byte[] readMessage(Stream clientStream)
        {
            if (cert2 == null)
                return readTcpMessage((NetworkStream)clientStream);
            else
                return readSslMessage((SslStream)clientStream);
        }

        private byte[] readSslMessage(SslStream clientStream)
        {
            using (System.IO.MemoryStream ms = new System.IO.MemoryStream())
            {
                byte[] buffer = new byte[2048];
                int bytes = -1;
                int offset = 0;
                int Waitcount = 0;

                while (client.Available == 0) ;

                do
                {
                    try
                    {
                        bytes = clientStream.Read(buffer, offset, buffer.Length);

                        ms.Write(buffer, 0, bytes);

                        if (Waitcount < 3)
                        {
                            Thread.Sleep(500);

                            Waitcount++;
                        }

                        offset += bytes;
                    }
                    catch
                    {
                        break;
                    }
                } while (bytes > 0 || Waitcount < 3);

                return ms.ToArray();
            }
        }

        private void HandshakeToClient(Stream clientStream, string message)
        {
            // process handshake (https://datatracker.ietf.org/doc/html/rfc6455#section-4.2.2)
            string swk = Regex.Match(message, "Sec-WebSocket-Key: (.*)").Groups[1].Value.Trim();
            string swka = swk + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
            byte[] swkaSha1 = System.Security.Cryptography.SHA1.Create().ComputeHash(Encoding.UTF8.GetBytes(swka));
            string swkaSha1Base64 = Convert.ToBase64String(swkaSha1);

            byte[] response = Encoding.UTF8.GetBytes(
                "HTTP/1.1 101 Switching Protocols" + Environment.NewLine +
                "Connection: Upgrade" + Environment.NewLine +
                "Upgrade: websocket" + Environment.NewLine +
                "Sec-WebSocket-Accept: " + swkaSha1Base64 +
                Environment.NewLine +
                Environment.NewLine);

            //요청 승인 응답 전송
            clientStream.Write(response, 0, response.Length);
        }

        private Stream getStream()
        {
            Stream stream = null;

                if (cert2 == null)
                    stream = client.GetStream();
                    /*
                else // sslstream
                    stream = getSslStream();
                     */

            return stream;
        }

        protected void WriteMessage(Stream clientStream, string message)
        {
            byte[] data = Encoding.UTF8.GetBytes(message);
            byte[] sendData;

            // firstByte 10000001 (fin=1, RSV1=0, RSV2=0, RSV3=0, opcode=0001(text)) 
            BitArray firstByte = new BitArray(new bool[] {
                    true,
                    false,
                    false,
                    false,
                    false,  //RSV3
                    false,  //RSV2
                    false,  //RSV1
                    true,   //Fin
                });

            if (data.Length < 126)
            {
                sendData = new byte[data.Length + 2];
                firstByte.CopyTo(sendData, 0);
                sendData[1] = (byte)data.Length;    //서버 -> 클라이언트는 mask하지 않는다.
                data.CopyTo(sendData, 2);
            }
            else
            {
                sendData = new byte[data.Length + 4];
                firstByte.CopyTo(sendData, 0);
                sendData[1] = 126;
                byte[] lengthData = new byte[2];
                lengthData[0] = (byte)((data.Length >> 8) & (byte)255);
                lengthData[1] = (byte)(data.Length & (byte)255);
                Array.Copy(lengthData, 0, sendData, 2, 2);
                data.CopyTo(sendData, 4);
            }

            clientStream.Write(sendData, 0, sendData.Length);  //클라이언트에 전송
            clientStream.Flush();
        }


    }
}
```