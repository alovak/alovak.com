---
title: "Mastering ISO 8583 Message Networking with Golang"
date: 2024-08-27
tags: engineering, payments, iso8583, golang, networking
layout: post
public: true
---
In the previous post, [Mastering ISO 8583 Messages](https://alovak.com/2024/08/15/mastering-iso-8583-messages-with-golang/), we looked at how to create specifications in Go, set data to messages, and finally, how to get binary values from them. This post will explore how messages are sent and received by card network participants such as acquirers and issuers. For the coding part to work with messages, we will use the [moov-io/iso8583](https://github.com/moov-io/iso8583/) Go package. The source code for the Go client we will be creating in this post can be found at the following link: [https://github.com/alovak/iso8583-client-demo](https://github.com/alovak/iso8583-client-demo).

## Payment transaction

ISO 8583 is a standard for financial transactions with payment cards. There are different types of financial transactions, and each transaction consists of request and response messages. Let's take a look at payment transactions. While the process of handling payment itself is more complex and includes clearing and settlement, in this document we focus only on the messages and the networking.

Here is a transaction flow for card authorization (this is when you pay for something, and your bank puts the amount on hold on your account). You can see how the request message flows from the acquirer to the card network, to the issuer, and then the response message is sent back to the acquirer:

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

<pre class="mermaid">
sequenceDiagram
  participant Seller
	participant Acquirer
	participant Card Network
	participant Issuer
	Seller->>Acquirer: Payment
	Acquirer->>Card Network: 0100
	Card Network->>Issuer: 0100
	Issuer-->>Issuer: authorizes request
	Issuer->>Card Network: 0110
	Card Network->>Acquirer: 0110
	Acquirer->>Seller: Confirmation
</pre>

What happens here is when an acquirer wants to charge a customer, it sends a 0100 authorization request message to the card network. The card network, based on the Primary Account Number (aka card number), routes the message to the card issuer.

The issuer checks the balance of the cardholder's account, deducts the amount of the transaction from the account, and puts it on hold. If these actions succeed, it replies to the card network with a 0110 authorization response message, which is sent back to the acquirer.

That’s good enough model for us to switch to the next mode advanced parts.

## Network connection

Card network participants who serve sellers/merchants and those who serve cardholders are all connected via card networks like VisaNet. POS devices, ATMs, and servers use various types of networks to facilitate these connections, with TCP/IP protocols being used in most cases to ensure reliable data transmission.

ISO 8583 specifications or related documents typically provide detailed information on establishing TCP/IP network connections to switches, routers, or gateways. *However, I have seen some exceptions, such as sending ISO 8583 messages over HTTP.*

In this post, we will focus on sending ISO 8583 messages over TCP/IP connections. Creating TCP connections is straightforward in Go using the [net](https://pkg.go.dev/net) package:

```go
conn, err := net.Dial("tcp", "1.2.3.4:20001")
if err != nil {
	// handle error
}
// you can write or read from the connection
```

## Network Messaging

With card networks, you usually get 2-4 (a limited number of) addresses/ports you can connect to. Then you keep the connections alive and use them to process all your load - hundreds or thousands of transactions.

Unlike HTTP (1.X), ISO 8583 messages do not work in a synchronous manner where you send a request and, while you are waiting for the response, the connection can't be used and is blocked.

Imagine that it may take a few seconds to get the transaction approval and response from the issuer. Having a limited number of connections would make transaction processing unacceptably slow. Have you ever waited for more than a couple of seconds to get approval for your payment in a restaurant or when paying online? You don't (excluding some incidents, of course). And that is because messages are processed asynchronously.

Two things help with asynchronous processing: message **Interleaving** and **binary framing**.

### Interleaving

An established TCP connection is a full-duplex channel - data can be transmitted in both directions simultaneously. It's like a road with two lanes, where traffic flows smoothly in both directions at the same time. Interleaving is when request messages are sent/written into the TCP connection one by one without waiting for the responses. Request messages can take different amounts of time to be handled - as some request messages have to reach servers on the opposite side of the earth or issuer servers are too busy - responses can be received in a different order than the requests were sent. As a result, response messages are sent back through the connection as soon as they are ready.

As you might see in the picture below, request messages 1-9 were sent/written to the connection, and only responses to messages 5 and 2 were received:


<img class="big" src="/images/iso8583-networking/interleaving.png" alt="message interleaving"/>

The question that arises here is if the order of requests and responses does not match, how do we know which response is for which request on the client? In the drawing above, each message has a number and type (res or req), so we need something like a message ID so we can match responses to requests. The Message Matching section of the ISO 8583 specification describes which fields of the request and response messages should be used for matching. 

> One of the fields that remains unchanged in the response message (echoed back) and can therefore be used as a message ID is the System Trace Audit Number, or **STAN** (field 11). While **STAN** is only 6 digits and cannot be unique across all transactions, it is sufficient to match messages within a short period of time.
> 

Please note that message matching is different from transaction matching. The former refers to matching a request with its corresponding response, while the latter involves matching different types of messages, such as a payment and a refund.

### Binary Framing

Let's look at how the sender and recipient see the data they write and read to/from the connection:

<img class="big" src="/images/iso8583-networking/binary-framing.png" alt="binary framing"/>

The sender writes messages into the TCP connection one by one, while the recipient reads the data stream and needs to know how to split that data into messages.

That's where the network length indicator (or header) is used. Each message is prepended with a length indicator containing the length of the message that the recipient should read from the connection:


<img class="big" src="/images/iso8583-networking/length-header.png" alt="length header"/>

The integration documentation from the card network will have information about the length of such an indicator and what exactly it contains. For example, it can be a 2-byte header indicating the length of the message, excluding the length of the header itself. There is a great [americanexpress/simplemli](https://github.com/americanexpress/simplemli) Go package that helps with many variants of length indicators.

> In some cases, a message may have not only a length header but also a message header, which contains information about the purpose of the message—whether it’s for session control (echo, shut down) or a regular message — as well as source and destination IDs, and other relevant details.
> 

## Implementing an ISO 8583 Network Client in Go

In this section, we will implement the ISO 8583 client that will allow us to send ISO 8583 messages to the server (card network, payment provider, etc.) and receive responses.

### Design

Before we start coding, let's think about what we are going to build. Imagine that you're building a payment service that accepts HTTP requests with payment information, and then you have to send these requests to the card network in ISO 8583 format.

An HTTP Server in Go handles each HTTP connection in its own goroutine, where for each HTTP request, an HTTP handler is called. For our imaginary payment service, HTTP handlers will convert request payloads into `*iso8583.Message` and, using the ISO 8583 client, send them to the destination server. I'll keep the conversion of JSON payload into the message out of scope. Here you can see how the system that uses the ISO 8583 client will work on a high level:

<img class="big" src="/images/iso8583-networking/design.png" alt="design"/>

From the developer's perspective, the ISO 8583 client needs a `Send` method that will accept an ISO 8583 request message as an argument and return a response message or an error. In addition, we want to be able to connect to and disconnect from the server. So, our public interface might look like this:

```go
type ISO8583Client interface {
  Send(message *iso8583.Message) (*iso8583.Message, error)
  Connect() error
	Close() error
}
```

Bear with me as I visualize the processes needed for message sending and receiving. They will both be based on [Goroutines](https://go.dev/tour/concurrency/1) and [channels](https://go.dev/tour/concurrency/2).

**Message sending**

The main idea here is that each `Send` method will send the request message into the requests channel and wait for the response message:


<img class="big" src="/images/iso8583-networking/write-loop.png" alt="write loop"/>

The write loop will do the following:

- Read request with message and response channel from the requests channel
- Pack the message into binary form
- Create length header
- Write length header and packed message into TCP connection
- Put response channel into the map using messageID as a key

**Message receiving**

The read loop will do the following:

- Read length header from the TCP connection
- Read message, knowing the message length from the length header
- Unpack the message
- Send message to the waiting sender

The `Send` method will wait for the response message to be received from the response channel.

<img class="big" src="/images/iso8583-networking/read-loop.png" alt="read loop"/>

**Message matching**

Finally, to match request messages to response messages, we will use a map where messageID (field 11 such as STAN will be used for it) will be a key and response channel will be a value. So, in the write loop, we will add messageID → response channel into the map, and in the read loop, we will look up the response channel based on the response message ID. I’m not sure how to draw it 🙈 and maybe this part will be easier to understand in code :)

### Code

The goal of the code below is to make all the concepts even clearer. Many things such as connection options, TLS, etc. are out of the scope of this document. We will keep only core aspects covered here. Think about it as a first draft of the ISO 8583 client, not as a production-ready solution.

Let’s define the type for our `Client`:

```go
type Client struct {
	// address is the address of the server
	address string

	// spec is the message spec we need to unpack the messages
	spec *iso8583.MessageSpec

	// conn is the network connection to the server (in our case it's TCP connection)
	conn net.Conn

	// requestsChan is the channel where the requests are sent to the write loop
	requestsChan chan Request

	// mu is the mutex that protects the pendingRequests map
	mu sync.Mutex

	// pendingResponses is the map that holds the pending responses
	// the key is the message ID (STAN in our case)
	pendingResponses map[string]chan Response
}
```

You can see that `requestsChan` is a channel where we will write instances of the `Request`, not the ISO 8583 messages. Let's define the `Request` and `Response`:

```go
type Request struct {
	message      *iso8583.Message

	// responseChan is the channel where the response is sent back to the caller
	responseChan chan Response
}

type Response struct {
	message *iso8583.Message
	err     error
}
```

The `Request` type holds the response channel. So, the write loop can put that response channel based on the message ID of the `message`.

Let's take a look at the `Connect` method:

```go
func (c *Client) Connect() error {
	slog.Info("connecting to the server")

	var err error

	// here you can configure the connection, for example, set the timeout
	// or set the keepalive options
	c.conn, err = net.Dial("tcp", c.address)
	if err != nil {
		return fmt.Errorf("connecting to %s: %w", c.address, err)
	}

	// start the write and read loops in separate goroutines to handle the requests and responses
	go c.writeLoop()
	go c.readLoop()

	return nil
}
```

For our example, we create a TCP connection to the address with default options for such a connection. In production, you might want to set TLS config, timeout, etc.

Here is our `Send` method:

```go
// Send sends the message to the server and waits for the response.
func (c *Client) Send(message *iso8583.Message) (*iso8583.Message, error) {
	// create a request
	request := Request{
		// set the message
		message: message,

		// create a channel where the response will be sent
		responseChan: make(chan Response),
	}

	// send the request to the write loop
	c.requestsChan <- request

	// wait for the response from the response channel or timeout
	select {
	case response := <-request.responseChan:
		// return the response to the caller of the Send method
		// it can be http handler or any other code that uses the client
		return response.message, response.err
	case <-time.After(5 * time.Second):
		// delete response channel from the map as we are not waiting
		// for the response anymore
		c.mu.Lock()
		delete(c.pendingResponses, messageID(message))
		c.mu.Unlock()

		// return an error if the timeout is reached
		return nil, fmt.Errorf("timeout waiting for the response")
	}
}
```

You can see that we create the `Request` with message and the response channel, then write it into the requests channel so the write loop can read it. Using the `select` statement, we wait for either a response to be received from the response channel or a timeout.

The beating heart of our client is the `writeLoop`:

```go
// it reads the request messages from the channel, packs them and sends them to the connection
// it stops when the channel is closed
// if there is an error sending the message, it closes the connection
// for the demo purpose, we will just log the errors
func (c *Client) writeLoop() {
	for request := range c.requestsChan {
		// pack the ISO 8583 message into bytes
		packed, err := request.message.Pack()
		if err != nil {
			// return error to the caller
			request.responseChan <- Response{nil, fmt.Errorf("error packing the message: %v", err)}
		}

		// let's create the 2 bytes length header
		lengthHeader := make([]byte, 2)
		binary.BigEndian.PutUint16(lengthHeader, uint16(len(packed)))

		// prepend the length header to the packed message
		packed = append(lengthHeader, packed...)

		// to avoid data races, as we will use the same map from the read loop
		// we need to lock the map
		c.mu.Lock()

		// save the response channel in the map using the message ID as the key
		c.pendingResponses[messageID(request.message)] = request.responseChan

		c.mu.Unlock()

		// write the header and packed message to the (TCP) connection
		_, err = c.conn.Write(packed)
		if err != nil {
			slog.Error("failed to send the message", "error", err)

			// return error to the caller
			request.responseChan <- Response{nil, fmt.Errorf("error sending the message: %v", err)}

			// close the connection
			c.Close()
		}
	}
}
```

As you can see in the loop, it:

- Reads requests from the requests channel
- Packs the message into bytes
- Creates length header
- Puts the response channel into the map (so later the read loop can find it)
- Writes it all into the TCP connection

Now, let’s dive into the `readLoop`:

```go
// it reads the response messages from the connection, unpacks them and sends
// them to the response channel
func (c *Client) readLoop() {
	for {
		// read the 2 bytes of length header
		lengthHeader := make([]byte, 2)

		_, err := c.conn.Read(lengthHeader)
		if err != nil {
			slog.Error("failed to read the message length header", "error", err)
			return
		}

		// convert the length header to uint16 to get the message length
		length := binary.BigEndian.Uint16(lengthHeader)

		// read the message into the slice of bytes with the length we got from the header
		rawMessage := make([]byte, length)
		_, err = c.conn.Read(rawMessage)

		slog.Info("received message", "message", fmt.Sprintf("%x", rawMessage))
		// unpack the message

		// create a new message using the spec and unpack the raw message
		message := iso8583.NewMessage(c.spec)
		err = message.Unpack(rawMessage)
		if err != nil {
			slog.Error("failed to unpack the message", "error", err)
			// we can continue with reading the next messages
			continue
		}

		// to avoid data races, as we will use the same map from the write loop
		// we need to lock the map
		c.mu.Lock()

		// get the response channel from the map using the message ID as the key
		responseChan, ok := c.pendingResponses[messageID(message)]
		if ok {
			// remove the response channel from the map as we don't need it anymore
			delete(c.pendingResponses, messageID(message))
		}

		// unlock the map
		c.mu.Unlock()

		// if the response channel is not found, log an error and continue with reading the next messages
		if !ok {
			slog.Error("received a response for an unknown message", "id", messageID(message))
			continue
		}

		// send the response to the caller's response channel
		responseChan <- Response{message, nil}
	}
}

```

from the code you see that it:

- Reads the length header
- Reads and unpacks the message
- Gets the response channel from the map using STAN as a message ID
- Sends the message to the response channel

Some edge cases you should be aware of related to reading messages:

> **Late response.** In production, you may encounter situations where a "late response" occurs. This happens when you've waited for a response for a set period (N seconds) and, after that time elapses, your system returns a timeout error to the caller of the `Send` method. However, a response might still arrive several seconds later—when no one is actively waiting for it.

and this one:

> **Server-Initiated Requests.** While our implementation focuses on sending requests and receiving responses, it’s important to note that the card network’s server can also send requests to the client. In reality, servers can send request messages, such as sign-off notifications (instructing you to stop sending messages over the connection), echo requests, and more. Therefore, the system should be prepared to handle incoming requests from the server as well.
> 

The final piece of the client shows how we get the message ID using STAN for it:

```go
// messageIDData is the struct that holds the message ID (STAN) field
type messageIDData struct {
	STAN string `iso8583:"11"`
}

// messageID extracts the message ID (STAN) from the message
func messageID(message *iso8583.Message) string {
	data := messageIDData{}
	err := message.Unmarshal(&data)
	if err != nil {
		return ""
	}

	return data.STAN
}
```

We unmarshal the message into the data struct with only one field - STAN, which is used as an ID for our demo.

I don’t want to focus on the `Close` method in this post, but it's crucial to implement it properly in a production environment. A graceful client disconnect should be implemented, taking the following steps into account:

- Stop accepting new messages when closing the client.
- Wait for all pending responses to be received.
- Finally, close the channels and connection, and ensure all goroutines have completed.

### Testing our implementation

Let's be sure that our client works. We need a test server we can connect to. For that purpose, we will create a simple TCP echo server that will also log received data. Here is the helper function to run and stop it:

```go
func startEchoServer(ctx context.Context) {
	ln, err := net.Listen("tcp", ":8080")
	if err != nil {
		slog.Error("Failed to listen on port 8080", "error", err)
		return
	}

	// Channel to notify goroutines to stop
	done := make(chan struct{})

	// close everything when the context is done
	// so the next gorooutine can return either because of the done channel or
	// the listener is closed
	go func() {
		<-ctx.Done()
		close(done)
		ln.Close()
		slog.Info("Server stopped")
	}()

	go func() {
		for {
			conn, err := ln.Accept()
			if err != nil {
				select {
				case <-done:
					return // Stop accepting new connections
				default:
					slog.Error("Failed to accept connection", "error", err)
					continue
				}
			}
			go handleConnection(conn)
		}
	}()
}

// just echo back the received data
// until the connection is closed
func handleConnection(conn net.Conn) {
	defer conn.Close()

	// Create a TeeReader that writes the data to a log while echoing it back
	reader := io.TeeReader(conn, logWriter{})

	// Copy the data from the reader (which logs it) back to the connection
	io.Copy(conn, reader)
}

// logWriter implements the io.Writer interface and logs the data it receives
type logWriter struct{}

func (logWriter) Write(p []byte) (n int, err error) {
	slog.Info("Server received (raw) request data", "data", fmt.Sprintf("%X", p))
	return len(p), nil
}
```

Using an echo server will work perfectly for our client since we use field 11 (STAN) for request and response matching. In production, we would also check MTI to ensure it’s the actual response `0110` to your `0100` request.

Now, let's write a test in which we start the server, connect to it, create a message, send it, and receive a response:

```go
func TestClient(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	// Start the echo server in the background until the test is done
	startEchoServer(ctx)
	defer cancel()

	// Create a new message using the spec
	requestMessage := iso8583.NewMessage(spec)

	// Set the message fields
	err := requestMessage.Marshal(&AuthorizationRequest{
		MTI:                 "0100",
		PAN:                 "4242424242424242",
		Amount:              1000,
		TransactionDatetime: time.Now().UTC().Format("060102150405"),
		Currency:            "840",
		CVV:                 "7890",
		ExpirationDate:      "2512",
		AcceptorInformation: &AcceptorInformation{
			MerchantName:         "Merchant Name",
			MerchantCategoryCode: "1234",
			MerchantPostalCode:   "1234567890",
			MerchantWebsite:      "https://www.merchant.com",
		},
		STAN: "000001",
	})
	require.NoError(t, err)

	// Create a new client
	c := client.New("localhost:8080", spec)

	// Connect to the server
	err = c.Connect()
	require.NoError(t, err)

	// Close the connection when the test is done
	defer c.Close()

	// Send the message to the server and wait for the response
	response, err := c.Send(requestMessage)
	require.NoError(t, err)
	require.NotNil(t, response)

	// let's print the response message to STDOUT
	iso8583.Describe(response, os.Stdout)
}
```

This is what we can see when we run the test:

```go
➜ go test
2024/08/25 15:36:24 INFO connecting to the server
2024/08/25 15:36:24 INFO Server received (raw) request data data=006B010073E000000000000031364242424242424242001000240825133624084078902512303636303131334D65726368616E74204E616D653032313233343033313031323334353637383930303430323468747470733A2F2F7777772E6D65726368616E742E636F6D000001
2024/08/25 15:36:24 INFO Client received (raw) response message message=010073e000000000000031364242424242424242001000240825133624084078902512303636303131334d65726368616e74204e616d653032313233343033313031323334353637383930303430323468747470733a2f2f7777772e6d65726368616e742e636f6d000001
ISO 8583 CardFlow Playgroud Specification Message:
MTI..........: 0100
Bitmap HEX...: 73E0000000000000
Bitmap bits..:
    [1-8]01110011    [9-16]11100000   [17-24]00000000   [25-32]00000000
  [33-40]00000000   [41-48]00000000   [49-56]00000000   [57-64]00000000
F0   Message Type Indicator.........: 0100
F2   Primary Account Number (PAN)...: 4242****4242
F3   Amount.........................: 1000
F4   Transmission Date & Time.......: 240825133624
F7   Currency.......................: 840
F8   Card Verification Value (CVV)..: 7890
F9   Card Expiration Date...........: 2512
F10  Acceptor Information SUBFIELDS:
-------------------------------------------
F01  Merchant Name.................: Merchant Name
F02  Merchant Category Code (MCC)..: 1234
F03  Merchant Postal Code..........: 1234567890
F04  Merchant Website..............: https://www.merchant.com
------------------------------------------
F11  Systems Trace Audit Number (STAN)..: 000001
2024/08/25 15:36:24 INFO closing the connection
PASS
```

From the output, you can see that:

- The client connected to the server
- The server received the request message and echoed it back
- The client received the response message
- The connection was closed

That’s it. You can find source code for this post here:  [github.com/card-flow/iso8583-client](http://github.com/card-flow/iso8583-client)

### Benchmarking

I’m curious how much time it will take to call `Send` method when we have a concurrent calls. To measure this we need a benchmark test:

```go
func BenchmarkClientSend(b *testing.B) {
	// disable logging for the benchmark
	slog.SetLogLoggerLevel(slog.LevelError)

	ctx, cancel := context.WithCancel(context.Background())
	// Start the echo server in the background until the benchmark is done
	startEchoServer(ctx)
	defer cancel()

	// Create a new client
	c := client.New("localhost:8080", spec)

	// Connect to the server
	err := c.Connect()
	require.NoError(b, err)

	// Close the connection when the benchmark is done
	defer c.Close()

	// Run the benchmark in parallel
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			// Create a new message using the spec
			requestMessage := iso8583.NewMessage(spec)

			// Incrementing STAN value
			stan := getStan()

			// Set the message fields
			err := requestMessage.Marshal(&AuthorizationRequest{
				MTI:                 "0100",
				PAN:                 "4242424242424242",
				Amount:              1000,
				TransactionDatetime: time.Now().UTC().Format("060102150405"),
				Currency:            "840",
				CVV:                 "7890",
				ExpirationDate:      "2512",
				AcceptorInformation: &AcceptorInformation{
					MerchantName:         "Merchant Name",
					MerchantCategoryCode: "1234",
					MerchantPostalCode:   "1234567890",
					MerchantWebsite:      "https://www.merchant.com",
				},
				STAN: stan,
			})
			require.NoError(b, err)

			// Send the message to the server and wait for the response
			response, err := c.Send(requestMessage)
			require.NoError(b, err)
			require.NotNil(b, response)
		}
	})
}
```

When I run it, I see these results:

```go
➜ go test -bench=BenchmarkClientSend -run=^\$ -cpu 6
goos: darwin
goarch: amd64
pkg: github.com/card-flow/iso8583-client/client
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkClientSend-6              22052             53706 ns/op
PASS
```

The benchmark shows that with 6 concurrent calls to the `Send` method, it took approximately 54 microseconds to send a message and receive the corresponding response. This time includes the full round-trip of sending the request, the server echoing back the response, and receiving that response on the client side.

### How to improve our implementation

Here is the battle-tested implementation of the [moov-io/iso8583-connection](https://github.com/moov-io/iso8583-connection), which started from code similar to what you've seen here. I recommend checking out its source code and tests. Over time, it has evolved into a mature, production-tested package that supports the following:

- Configuration options
- TLS connections
- Inbound message handlers
- Handling of unmatched messages
- Graceful closing of the connection with waiting for all pending messages to be handled
- Reply to message as you not always want to send and get a response
- Connection pool with automatic reconnect
- Allow to use connection on the server side (in the end, it's just a connection between two parties)
- Have a server implementation (useful for building test servers)
- And so on.

## Summary

In this blog post we explored the networking aspects of ISO 8583 messages, focusing on how these messages are transmitted and received by participants in card networks. Key points covered include:

1. An overview of payment transaction flow
2. Explanation of network messaging concepts, including message interleaving and binary framing, which enable asynchronous processing of transactions
3. A detailed implementation of an ISO 8583 network client in Go
4. Testing the implementation using a simple TCP echo server.

The post offers both theoretical knowledge about ISO 8583 networking and practical implementation guidance. I hope it will be helpful to developers working on payment systems or those interested in understanding financial transaction protocols in depth.

If you found this document helpful, or if you have any questions or suggestions, I’d love to hear from you! You can reach out to me on [LinkedIn](https://www.linkedin.com/in/pavelgabriel/) or find me in the [#iso8583](https://moov-io.slack.com/archives/C014UT7C3ST) channel of the Moov Financial community Slack.

