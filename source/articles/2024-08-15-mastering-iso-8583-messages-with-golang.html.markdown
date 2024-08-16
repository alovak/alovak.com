---
title: "Mastering ISO 8583 messages with Golang"
date: 2024-08-15
tags: engineering, payments, iso8583, golang
layout: post
public: true
---
        
As the author and maintainer of the [moov-io/iso8583](https://github.com/moov-io/iso8583) package, who has built direct integrations and passed certifications with most card networks (Visa, Mastercard, Discover and Amex), I want to share my hands-on experience with other engineers. If you're curious about how card processing works behind the scenes and want to explore it from a code perspective, this guide is for you.

When you integrate with card networks or payment providers using ISO 8583, you typically receive a hefty PDF document, often hundreds of pages long,  that outlines message flows, message format, how fields are encoded, the purpose of each field, and more. Understanding these specifications and generating correct messages can be a daunting task.

The goal of this guide is to show you how to convert an ISO 8583 **message specification** into Go code and how to build and parse binary messages using the [moov-io/iso8583](https://github.com/moov-io/iso8583/) package. You'll also see how to validate the correctness of the specification Go code and how to troubleshoot issues. Aspects such as message flows, message or network headers, TCP connections, etc., are beyond the scope of this document. 

If you're the "show me the code" person, then you can find the test with all the code snippets from this guide right [here](https://github.com/alovak/cardflow-playground/blob/main/examples/message_test.go).

## ISO 8583 Messages

An ISO 8583 message layout contains the following:

- **MTI** - message type indicator
- **Bitmap** - one, two, or three bitmaps whose bits indicate which fields are present in the message
- **Message fields (Data elements)** - fields defined by the ISO 8583 standard, such as processing code, transaction amount, processing time, acceptor information, and so on

ISO 8583 defines the message fields and sub-fields. There are different versions and revisions of the standard—1987, 1993, 2003, 2023. Each card network and provider may use some fields for their own “private” purposes. Field values can be encoded with different encodings such as ASCII, EBCDIC, BCD, Hex. Some fields may have a length indicator, which can also be encoded in various ways. Such variability makes a developer's life more… interesting when they have to build an integration.

As an engineer, I see ISO 8583 messages as being described similarly to [Thrift](https://thrift.apache.org/), [Protocol Buffers](https://code.google.com/p/protobuf/), or [Avro](https://avro.apache.org/). In this document, I want to focus on the schema nature of ISO 8583, not on the standard fields. So, in the next section, you’ll see a toy message specification. It doesn’t follow the standard, as there are more than a hundred fields; instead, it includes some fields that, in my opinion, make sense for such a demo schema. I use this toy spec in the [CardFlow Playground](https://github.com/alovak/cardflow-playground) project to show how acquiring and issuing work.

*Please note that the field numbers in the toy specification don’t match the ISO 8583 standard.*

## **ISO 8583 Playground Specification**

Each field of the message can be encoded using one of the encodings described below. If field has varied length, then there should be a length prefix before it.

### **Encodings**

- **ASCII (American Standard Code for Information Interchange):**
    This encoding is used for alphanumeric data fields, where each character is represented by an 8-bit binary number. ASCII encoding is used for fields such as the Merchant Name and Website.
    
- **BCD (Binary-Coded Decimal):**
    This encoding is used for numeric data fields, where each decimal digit (0-9) is represented by a 4-bit binary sequence. BCD is typically used in fields such as the Message Type Indicator (MTI), Amount, and Card Expiration Date. Encoded values should be right-justified with leading zeros.
    
- **Binary:**
    This encoding is used for fields like the Bitmap or any other fields that contain non-printable data.
    

### **Variable Length Fields**

- **L (1-digit length prefix):**
    This prefix indicates that the length of the data field is defined by a single ASCII-encoded digit (0-9) that follows the prefix. It is not used in this specific message specification.
    
- **LL (2-digit length prefix):**
    This prefix indicates that the length of the data field is defined by two ASCII-encoded digits (00-99) that follow the prefix. In this specification, it is used for fields like the Primary Account Number (PAN) and certain subfields within the Acceptor Information.
    
- **LLL (3-digit length prefix):**
    This prefix indicates that the length of the data field is defined by three ASCII-encoded digits (000-999) that follow the prefix. It is used for the composite Acceptor Information field.
    

### **Fields in the 0100 (Authorization Request) and 0110 (Authorization Response) Messages**

<div class="small-table"></div>

|  | Field Name | Description | Length | Encoding | Request/Response | Example Value |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | MTI | Message Type Indicator | 4 Fixed | BCD | Both | 0100 / 0110 |
| 1 | Bitmap | Bitmap indicating presence of data elements | 8 Fixed | Binary | Both | 7FFFFFFFFFFF0000 |
| 2 | Primary Account Number (PAN) | The cardholder's account number | Up to 19 Variable (LL) | BCD | Request | 1234567890123456 |
| 3 | Amount | Transaction amount in the smallest currency unit. The value is right justified with leading zeroes. | 6 Fixed | BCD | Both | 000100 (for 1.00) |
| 4 | Transmission Date & Time | The date and time the message was sent | 20 Fixed | BCD | Both | 20230810123456 |
| 5 | Approval Code | Code for approving the transaction | 2 Fixed | BCD | Response | 00 |
| 6 | Authorization Code | Code for authorizing the transaction | 6 Fixed | BCD | Response | 123456 |
| 7 | Currency | Transaction currency code | 3 Fixed | BCD | Both | 840 (for USD) |
| 8 | Card Verification Value (CVV) | Card security code | 4 Fixed | BCD | Request | 1234 |
| 9 | Card Expiration Date | The card's expiration date | 4 Fixed | BCD | Request | 2512 |
| 10 | Acceptor Information | Composite field with details about the acceptor/merchant | Up to 999 Variable (LLL) | ASCII | Request | (See Below) |
| 11 | STAN | Systems Trace Audit Number, a unique identifier for the transaction | 6 Fixed | BCD | Both | 654321 |

### **Acceptor Information Composite Field**

The Acceptor Information field is a composite field with subfields, each identified by a 2-digit tag. The composite field is prefixed by a 3-digit length encoded in ASCII (LLL prefix).

**Subfields:**

<div class="small-table"></div>

| Tag | Field Name | Description | Length/Type | Encoding | Prefix | Example Value |
| --- | --- | --- | --- | --- | --- | --- |
| 01 | Merchant Name | Name of the merchant | Up to 99 Variable | ASCII | LL | "ACME Corp" |
| 02 | Merchant Category Code (MCC) | Category code of the merchant | 4 Fixed | ASCII | None | 1234 |
| 03 | Merchant Postal Code | Postal code of the merchant | Up to 10 Variable | ASCII | LL | 12345 |
| 04 | Merchant Website | Website of the merchant | Up to 299 Variable | ASCII | LLL | www.acme.com |

## Creating the ISO 8583 Spec in Go

Now, let’s create a Golang spec using the [moov-io/iso8583](https://github.com/moov-io/iso8583/) project.

```go
var spec *iso8583.MessageSpec = &iso8583.MessageSpec{
	Name: "ISO 8583 CardFlow Playgroud ASCII Specification",
	Fields: map[int]field.Field{
		0: field.NewString(&field.Spec{
			Length:      4,
			Description: "Message Type Indicator",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		1: field.NewBitmap(&field.Spec{
			Length:      8,
			Description: "Bitmap",
			Enc:         encoding.Binary,
			Pref:        prefix.Binary.Fixed,
		}),
		2: field.NewString(&field.Spec{
			Length:      19,
			Description: "Primary Account Number (PAN)",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.LL,
		}),
		3: field.NewString(&field.Spec{
			Length:      6,
			Description: "Amount",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
			Pad:         padding.Left('0'),
		}),
		4: field.NewString(&field.Spec{
			Length:      20,
			Description: "Transmission Date & Time",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		5: field.NewString(&field.Spec{
			Length:      2,
			Description: "Approval Code",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		6: field.NewString(&field.Spec{
			Length:      6,
			Description: "Authorization Code",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		7: field.NewString(&field.Spec{
			Length:      3,
			Description: "Currency",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		8: field.NewString(&field.Spec{
			Length:      4,
			Description: "Card Verification Value (CVV)",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		9: field.NewString(&field.Spec{
			Length:      4,
			Description: "Card Expiration Date",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
		10: field.NewComposite(&field.Spec{
			Length:      999,
			Description: "Acceptor Information",
			Pref:        prefix.ASCII.LLL,
			Tag: &field.TagSpec{
				Length: 2,
				Enc:    encoding.ASCII,
				Sort:   sort.StringsByInt,
			},
			Subfields: map[string]field.Field{
				"01": field.NewString(&field.Spec{
					Length:      99,
					Description: "Merchant Name",
					Enc:         encoding.ASCII,
					Pref:        prefix.ASCII.LL,
				}),
				"02": field.NewString(&field.Spec{
					Length:      4,
					Description: "Merchant Category Code (MCC)",
					Enc:         encoding.ASCII,
					Pref:        prefix.ASCII.Fixed,
				}),
				"03": field.NewString(&field.Spec{
					Length:      10,
					Description: "Merchant Postal Code",
					Enc:         encoding.ASCII,
					Pref:        prefix.ASCII.LL,
				}),
				"04": field.NewString(&field.Spec{
					Length:      299,
					Description: "Merchant Website",
					Enc:         encoding.ASCII,
					Pref:        prefix.ASCII.LLL,
				}),
			},
		}),
		11: field.NewString(&field.Spec{
			Length:      6,
			Description: "Systems Trace Audit Number (STAN)",
			Enc:         encoding.BCD,
			Pref:        prefix.BCD.Fixed,
		}),
	},
}
```

You see that our spec is a map of `field.Field` types with `field.Spec` where we set the following:

- **Length** - the length of the field value; if it's a variable-length field, then it sets the max length of the field value
- **Enc** - how the field value should be encoded/decoded when we pack/unpack it
- **Pref** - used to add the length indicator (prefix) before the field value. We can specify its encoding (in our example, BCD or ASCII) and the length of the prefix—L, LL, LLL (one digit, two, or three). A fixed prefix means that no length indicator will be added, as we know the length of the field from the schema/specification.
- **Pad** - uses a character to pad the value on the right or left.

As you can see, for field 10, which includes subfields with tags (`01`, `02`, and so on), we use `field.NewComposite`. This type of field can be seen as an embedded message. It can have its own bitmap, fields with subfields, tags, etc.

> 💡 **The best practice** when working with the [moov-io/iso8583](https://github.com/moov-io/iso8583/) package is to **have one specification that defines all fields of the spec**, not just the fields you need in the request or response message. If you define only a subset of fields, the parser will fail to parse a message if it receives fields not described in the spec. Also, if you plan to use the [moov-io/iso8583-connection](https://github.com/moov-io/iso8583-connection) package for networking, it will require a single specification that applies to all messages.

In the next section, you will see how we use one spec but set only the message fields we need for our request.

> 💡 If you don’t want to define some composite fields, you can use a trick by defining the field as `field.Binary`. In this case, you will just read the whole value of the field without parsing it.

## Message Packing and Unpacking

I’ll write the code in the form of a Go test to be able to run it and see if it works as expected. Doing it this way shows the intent and highlights important aspects of the code.

We have a specification in Go code. Now, let’s create a message with the data for the authorization request. First, we have to define the data struct—the type that will contain the field values we want to send. Then, we should fill in the message with these values using the `Marshal` method of the message. Here’s how it should be done:

```go
// We use field tags to map the struct fields to the ISO 8583 fields
type AcceptorInformation struct {
    MerchantName         string `iso8583:"01"`
    MerchantCategoryCode string `iso8583:"02"`
    MerchantPostalCode   string `iso8583:"03"`
    MerchantWebsite      string `iso8583:"04"`
}

type AuthorizationRequest struct {
    MTI                 string               `iso8583:"0"`
    PAN                 string               `iso8583:"2"`
    Amount              int64                `iso8583:"3"`
    TransactionDatetime string               `iso8583:"4"`
    Currency            string               `iso8583:"7"`
    CVV                 string               `iso8583:"8"`
    ExpirationDate      string               `iso8583:"9"`
    AcceptorInformation *AcceptorInformation `iso8583:"10"`
    STAN                string               `iso8583:"11"`
}

// Create a new message
requestMessage := iso8583.NewMessage(spec)

// Set the message fields
err := requestMessage.Marshal(&AuthorizationRequest{
    MTI:                 "0100",
    PAN:                 "4242424242424242",
    Amount:              1000,
    TransactionDatetime: time.Now().Format("060102150405"),
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

// Pack the message
packed, err := requestMessage.Pack()
require.NoError(t, err)

```

As a result of the `Pack` method, we have the packed (binary - slice of bytes) value of the message in the `packed` variable.

As a best practice, I recommend doing the opposite action in the test—unpacking the `packed` value and then comparing field values in the data structs. Take a look:

```go
// Unpack the message
responseMessage := iso8583.NewMessage(spec)
err = responseMessage.Unpack(packed)
require.NoError(t, err)

// Unmarshal the message fields
var authorizationRequest AuthorizationRequest
err = responseMessage.Unmarshal(&authorizationRequest)
require.NoError(t, err)

// Check the message fields
require.Equal(t, "0100", authorizationRequest.MTI)
require.Equal(t, "4242424242424242", authorizationRequest.PAN)
require.Equal(t, int64(1000), authorizationRequest.Amount)
require.Equal(t, time.Now().Format("060102150405"), authorizationRequest.TransactionDatetime)
require.Equal(t, "840", authorizationRequest.Currency)
require.Equal(t, "7890", authorizationRequest.CVV)
require.Equal(t, "2512", authorizationRequest.ExpirationDate)
require.Equal(t, "Merchant Name", authorizationRequest.AcceptorInformation.MerchantName)
require.Equal(t, "1234", authorizationRequest.AcceptorInformation.MerchantCategoryCode)
require.Equal(t, "1234567890", authorizationRequest.AcceptorInformation.MerchantPostalCode)
require.Equal(t, "https://www.merchant.com", authorizationRequest.AcceptorInformation.MerchantWebsite)
require.Equal(t, "000001", authorizationRequest.STAN)
```

By doing so, we can be sure that our specification works in both directions and that there are no differences in the field values or binary values. I’ve encountered rare cases where discrepancies might occur, especially when using custom code or expecting padded values. Having such a test removes any doubts.

## Checking the Correctness of the Spec

We were able to pack and unpack the message with our spec. But how do we know that the packed values it produces are valid and what the card network expects to receive from us? This is when having an example message from your provider or card network is priceless.

> 💡 Having example messages can significantly simplify and speed up the process of creating the correct specification.

For instance, if the spec indicates that a field has a maximum length of 256 characters and a 1-byte length indicator (prefix), it’s not always clear whether the prefix is included in the length.

My experience has taught me that there are countless variations, and the only reliable way to navigate them is by using a validator of some sort. I’ll show you how to use an example message to validate the correctness of your specification.

Let’s assume we’ve been given an example of a 0100 message in a file. If I open the file with a text editor, I might see unreadable garbage like this:

```
sBBBBBBBB@x%0660113Merchant Name0212340310123456789004024https://www.merchant.com
```

It’s important to remember that ISO 8583 messages are in binary form, and in cases where all fields are not in ASCII encoding, you won’t be able to see and recognize all values. For that reason, to allow you to see each byte of the message, its value is encoded in HEX:

```
010073E000000000000031364242424242424242001000240812160140084078902512303636303131334D65726368616E74204E616D653032313233343033313031323334353637383930303430323468747470733A2F2F7777772E6D65726368616E742E636F6D000001
```

Keep in mind, messages are not sent in HEX format—they are sent as binary values. HEX is used only for readability (kind of).

Having a HEX representation, we can update our test with this:

```go
// Here is the example of the packed message
examplePackedMessage := "010073E000000000000031364242424242424242001000240812160140084078902512303636303131334D65726368616E74204E616D653032313233343033313031323334353637383930303430323468747470733A2F2F7777772E6D65726368616E742E636F6D000001"

// Check the packed message
require.Equal(t, examplePackedMessage, hex.EncodeToString(packed))
```

It’s a good moment to stop, but if you run the test now, it will fail. I want to show you how to find the reason and fix the issue. That will give you an even better understanding of how an ISO 8583 message is built.

## Troubleshooting

If I run the test, it fails with an error that the packed value doesn’t match the example value (please, ignore that the values are in different cases). My next step is to try to see the value of each field and compare them visually (you can also unpack into a new struct and compare struct, but the point here is to show you the message source). This is a visual inspection, which is tricky, but sometimes it’s the only way to find what’s wrong with the spec. Here’s what I see and how I find the error:

<img src="/images/inspect-iso-8583-message-2.png" alt="block cipher encryption" style="width: 710px;"/>

I see the values of MTI, Bitmap, PAN LL prefix, and then PAN, Amount, and Processing Date & Time, and so on.

Let’s take a closer look at some fields we see. Remember, we see a Hex encoded value, where each byte is represented by a Hex digit (consisting of two chars or nibbles). `LL` prefix with value `3136` for the PAN field. Our variable length prefixes are encoded in ASCII, so let’s convert the `3136` value into ASCII:

- 31 (Hex) → 49 (Byte) → 1 (ASCII char)
- 36 (Hex) → 54 (Byte) → 6 (ASCII char)

So, we have `16` as the value of the length indicator, which means that the PAN length is 16 digits.

You can notice that the Processing Date & Time fields do not match. That’s because in our test we use `time.Now` and in the example message, we have a fixed time (in the past). So, we should fix our test by setting the time to match the time in the example message. In real situations, you may have to update more fields.

Let’s update our test to make it pass:

```go
// use time from our example
timeFromExample := "240812160140"
processingTime, err := time.Parse("060102150405", timeFromExample)
require.NoError(t, err)

// Set the message fields
err = requestMessage.Marshal(&AuthorizationRequest{
    MTI:                 "0100",
    PAN:                 "4242424242424242",
    Amount:              1000,
    TransactionDatetime: processingTime.Format("060102150405"),
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
```

With this update, our test passes. Here you can find the final version of the test and play with it.

Also, there is a handy `iso8583.Describe` helper that allows you to inspect message fields. Let’s take a look at what you can see using it. We will add the following to our test:

```go
// output the message content into Stdout
err = iso8583.Describe(requestMessage, os.Stdout)
require.NoError(t, err)

```

It prints the following to the STDOUT:

```bash
➜ go test
ISO 8583 CardFlow Playgroud ASCII Specification Message:
MTI..........: 0100
Bitmap HEX...: 73E0000000000000
Bitmap bits..:
    [1-8]01110011    [9-16]11100000   [17-24]00000000   [25-32]00000000
  [33-40]00000000   [41-48]00000000   [49-56]00000000   [57-64]00000000
F0   Message Type Indicator.........: 0100
F2   Primary Account Number (PAN)...: 4242****4242
F3   Amount.........................: 1000
F4   Transmission Date & Time.......: 240812160140
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
PASS
ok      github.com/alovak/cardflow-playground/examples  1.031s
```

What a nice structured view! You can output it into the log or any `io.Writer` and store it if you need. As you can see, the helper filters some known fields (such as PAN) by default. You can also pass a custom filter to the `Describe` function to filter out fields you don’t want to see. Here is how you can do it:

```go
// to make it right, let's filter the value of CVV field when we output it
filterCVV := iso8583.FilterField("8", iso8583.FilterFunc(func(in string, data field.Field) string {
    if len(in) == 0 {
        return in
    }
    return in[0:1] + strings.Repeat("*", len(in)-1)
}))

// don't forget to apply default filter
filters := append(iso8583.DefaultFilters(), filterCVV)

err = iso8583.Describe(requestMessage, os.Stdout, filters...)
require.NoError(t, err)
```

Now, the output for the CVV filed will look like this:

```bash
F8   Card Verification Value (CVV)..: 7***
```

## Summary

Getting the hang of ISO 8583 messaging in Golang can seem pretty daunting at first, but with the right approach, it's totally doable. In this article, we broke down how to take an ISO 8583 spec and turn it into Golang code, build and parse those tricky binary messages, and even validate your work against examples. With the iso8583 Golang package, along with some practical tips on testing and troubleshooting, you’re well-equipped to tackle ISO 8583 in your Golang projects.

Feel free to check out the tests in the [moov-io/iso8583](https://github.com/moov-io/iso8583/) package for more advanced examples or even contribute by posting issues or PRs on [GitHub](https://github.com/moov-io/iso8583).

If you found this guide helpful, or if you have any questions or suggestions, I’d love to hear from you! You can reach out to me on [LinkedIn](https://www.linkedin.com/in/pavelgabriel/) or find me in the [#iso8583](https://moov-io.slack.com/archives/C014UT7C3ST) channel of the Moov Financial community Slack.

