# Building ping command in Node.js for fun. Short guide to Wireshark, buffers, sockets, and bit manipulation

I find it really helpful to sometimes explore some unknown things to have better understanding how system works. You probably used `ping` command multiple times to test connection to the server. This is widely available tool. Do you know how it works?

In this article, I'll share my learning on how to implement some of the low-level binary protocols using ping as an example. TODO: We will cover some points on how it can be implemented in node.js.

## What is ping?

Let's start from the very beginning what is ping command. It is [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) protocol that can be used for system diagnostic. This is not TCP/UDP protocol that is also based on IP, but something working alongside of it. So it works slightly different and there are no ports to connect to.

You can find protocol description on [wiki](<https://en.wikipedia.org/wiki/Ping_(networking_utility)#Message_format>)

Full IPv4 message
![image](./imgs/request-full-ipv4-datagram.png)

Here most of the work will be done automatically for us by OS or socket library. However, all red parts we will have to implement ourselves.

ICMP message

![image](./imgs/request-datagram.png)

For simplicity, I'll use IPv4 and ICMP, however, ICMPv6 works almost exactly the same.

## Test it/debug it/Wireshark it!

So now we have good theory behind of us to get started. However, in order to debug our program it would be helpful to see actual bits and bytes we send to some IP. The easiest way to check ANY traffic that your computer is serving is to use [Wireshark](https://www.wireshark.org/download.html). Before we start let's start with checking if theory and practice align on how protocol works.

Two easy steps:

1. Start Wireshark and choose what to capture (in my case wifi that is connected to the Internet)
2. Add filter icmp in the top bar as it will produce a lot of noise

This is more or less what you will see

![image](./imgs/wireshark.png)

Wireshark is quite powerful and knows how to handle ICMP commands. It means it will be very helpful to validate our setup. You can also hover over Packet Bytes to highlight what are they responsible for. In this case you can see Type of request is equal 8.

## Let's code

Now that we have all building blocks in place we can start coding. Unfortunately, node.js has not native support for raw sockets to use ICMP protocol. However, there is a `raw-socket` npm package that adds support for it using [node-gyp](https://github.com/nodejs/node-gyp) to access system level APIs.

What we will need to do is listen and send data using our ICMP socket. So, let's get started

```js
import raw from "raw-socket";

const socket = raw.createSocket({ protocol: raw.Protocol.ICMP });

socket.on("message", function (buffer, source) {
  // TODO Decode
});

socket.on("error", (e) => {
  console.error("Socket Error: ", e);
  socket.close();
});

const pingBuffer = createPingBuffer(1234, 0, "Payload");
socket.send(pingBuffer, 0, buffer.length, "1.1.1.1", function (error, bytes) {
  if (error) console.error("Unable to send message: ", error.toString());
});

function createPingBuffer(identifier, sequence, payload) {
  // TODO Encode
}
```

This is main part of the app. Here we create ICMP socket and start listening for new messages. At the same time we send our ping command to `1.1.1.1` IP address.

So now we need to implement encoding and decoding of the messages

### Encoding ping protocol

As you saw in the diagram above we have to provide 8 bytes of information Type, Code, Checksum, Identifier, and Sequence Number. We can also provide optional payload that will be returned to us in the response.

So minimal ping request will look as following

![image](./imgs/requet-expected-minimal.png)

```
Type = 8 (Request) = 00001000 (binary)
Code = 0 = 00000000 (binary)
Checksum = dynamic value
Identifier = 1 = 00000001 (binary)
Sequence Number = 0 = 00000000 (binary)
```

In order to manipulate bits we will need to use Node.js [Buffers](https://nodejs.org/api/buffer.html). It is subclass of JavaScript's Uint8Array and adds helpful methods to manipulate bytes.

```js
const ICMP_HEADER_SIZE = 8;

function createPingBuffer(identifier, sequenceNumber, payload) {
  const buffer = Buffer.alloc(ICMP_HEADER_SIZE);

  buffer.writeUInt8(8, 0); // Type 1 byte with offset 0 bytes
  buffer.writeUInt8(0, 1); // Code 1 byte with offset 1 byte
  buffer.writeUInt16BE(0, 2); // Checksum 2 bytes with offset 2 bytes
  buffer.writeUInt16BE(identifier, 4); // Identifier 2 bytes with offset 4 bytes
  buffer.writeUInt16BE(sequenceNumber, 6); // Sequence Number 2 bytes with offset 6 bytes

  // Overrite 0 checksum with correct value based on full request
  raw.writeChecksum(buffer, 2, raw.createChecksum(buffer));

  return buffer;
}
```

That is it! We should be able to send ping request now. Let's check if it actually works using Wireshark.

![image](./imgs/wireshark-request.png)

As you can see it works as expected and we were able to send request. However, we still need to get the response

### Decode ping protocol

To decode ping protocol we can use similar helper methods available on Buffer class. However there is a caveat. We will get full IP protocol packet as a response. However, IP protocol has dynamic header size. It means that we will have to find where exactly our ping message starts.

Let's take a closer look to datagram

![image](./imgs/ip-header.png)

IHL stands for [Internet Header Length ](https://en.wikipedia.org/wiki/Internet_Protocol_version_4#IHL). It has 4 bits that specify the number of 32-bit (4 bytes) words in the header. Unfortunately, node.js Buffers don't have any helper methods to read 4 bit data. It means that we will have to implement it ourselves.

Here we will need to use [bitwise operations](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_AND) In order to get last 4 bits of a byte we will need to use bitwise and

```js
function getIPProtocolHeaderSize(buffer) {
  const versionAndIHL = buffer.readInt8();
  const IHL = versionAndIHL & 0b00001111; // will remove everything in the first 4 bits -> 0x0000XXXX
  return IHL * 4; // multiply by 4 as it contains number of 32-bit (4 bytes) words
}
```

Now we have everything we need to read response

```js
function toProtocolObject(buffer) {
  const ipOffset = getIPProtocolHeaderSize(buffer);

  // IP level TTL
  const ttl = buffer.readUInt8(8);

  const type = buffer.readUInt8(ipOffset);
  const code = buffer.readUInt8(ipOffset + 1);
  const checksum = buffer.readUInt16BE(ipOffset + 2);
  const identifier = buffer.readUInt16BE(ipOffset + 4);
  const sequenceNumber = buffer.readUInt16BE(ipOffset + 6);

  return {
    ttl,
    type,
    code,
    checksum,
    identifier,
    sequenceNumber,
  };
}
```

That is it! Now we just need to precisely measure time between request and response and we got fully functional ping client.

### Time measurement

If you actually pay closer attention to the original ping implementation you will notice a special trick how to make our client stateless and don't require us to track state for request/response time. As ping protocol requires to send us back the same payload we can send time as part of the payload! Unix implementation of ping is open source so we can learn some tricks from reading [source code](https://github.com/dspinellis/unix-history-repo/blob/BSD-4_3/usr/src/etc/ping.c#L233)

Let's use the same approach in our client. We will send precise time when we send command and compare it when we get the message. To get accurate time we can use [Performance](https://nodejs.org/api/globals.html#performance) API to do so

```js
const time = performance.timeOrigin + performance.now();

const uint32 = new Uint32Array(2);
uint32[0] = time / 1000; // seconds part
uint32[1] = (time - uint32[0] * 1000) * 1000; // hack to convert to microseconds that original ping uses

// write it as first 8 bytes of payload
buffer.writeUInt32BE(uint32[0], 8);
buffer.writeUInt32BE(uint32[1], 12);
```

## Conclusion
