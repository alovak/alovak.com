---
title: "ISO 8583 & Golang: Mastering Card Acquiring and Issuing"
date: 2023-04-27
tags: engineering, payments, iso8583, golang
layout: post
public: false
---

# Intro

As a seasoned engineer in the fintech industry with over 15 years of experience, I've designed and built various payment gateways, integrating them with numerous payment systems using high-level APIs. While APIs like Stripe, CyberSource or Mastercard Internet Gateway offered limited insight into the inner workings of these systems, my natural curiosity urged me to find a way to peek inside. Finally, at Moov, I had the unique opportunity to fulfill my geeky dream of understanding the core operations of these systems by helping to develop end-to-end card acquiring and issuing services with direct integration to major card brands.

With the goal of helping other engineers who may be interested in learning about acquiring, issuing, and potentially building solutions within these domains, I created the [CardFlow Playground](https://github.com/alovak/cardflow-playground) project using Golang and some of Moov's open source packages to explain it all.

# Overview of Card Authorization

## Card Authorization Process

When you make a payment at a grocery store or an online shop, it's easy to think of it as a simple, atomic operation - a payment. In reality, it's a multi-step process, with the first stage being the card authorization transaction.

Card authorization is required whenever a card is used for a payment. This process involves sending card data from the point of sale (POS) through a chain of services and providers to the bank that issued the card, ultimately to approve or decline the transaction.

The key participants in this process include the cardholder, the merchant (who accepts the card as a payment method), the acquirer, the payment network (such as Visa or Mastercard), and the issuing bank.

## Participants

There are numerous participants involved in the card authorization flow, each playing a specific role in the process:

1. [Merchant](https://moov.io/resources/dictionary/#merchant): The seller who accepts card payments for goods or services.
2. [Acquirer](https://moov.io/resources/dictionary/#acquirer--acquiring-bank) - Manages relationships with merchants, including onboarding, providing POS devices, APIs, and other necessary services. The acquirer is also connected to the payment network, which allows it to pass authorization requests and later pull money from the issuer during the settlement process.
3. **Payment Network**: Connects acquirers and issuers, facilitating the authorization, routing the requests and later settlement of transactions. Examples include Visa and Mastercard.
4. [Issuer](https://moov.io/resources/dictionary/#issuer) The bank or service that issued the card to the cardholder. The issuer manages relationships with cardholders, providing customer support, protecting them from fraud, and handling other account-related services.

It's important to note that there are many more participants involved in the card authorization flow than those mentioned in this article. 

However, for the sake of conciseness and to keep our focus on the CardFlow Playground project, we will simplify the scenario by working with a single acquirer and a single issuer. 

# **ISO 8583: A Common Language for Financial Systems**

ISO 8583 is designed to be a common language that financial systems use to communicate with each other when handling card transactions. It helps everyone in the card authorization flow—acquirers, issuers, and other players—understand each other by establishing a consistent messaging format, data elements, and communication flow.

## ISO 8583 Message Structure

An ISO 8583 message consists of a Message Type Indicator (MTI), bitmap, and a set of fields. The MTI is a type of transaction, such as authorization or reversal. It's a four-digit code where each digit has its own meaning. For example, `0100` is an authorization request, and `0110` is a response to that request.

Unlike JSON message, ISO 8583 message has no schema. Having message it’s hard to see  what’s is inside as you get a bunch of bytes. To know what's inside, you need a schema/specification that describes all possible fields, their length, and format. Message bitmap indicates which fields are in the received message. Each bit in the bitmap signifies the presence or absence of the field in the message. For example, if bit 5 is set, then field 5, as described in the specification, is in the message. 

The interesting thing about ISO 8583 is that each field can represent a set of fields with its own bitmaps and so on. It's like a Matryoshka doll of fields, endlessly nested, just waiting to surprise you with even more fields!

## Network Management: Binary Framing and Interleaving

When acquirers and issuers communicate with payment networks using ISO 8583 messages, there are a couple of things to be aware of.

In most cases, persistent TCP/IP socket connection is established between sender and receiver of ISO 8583 messages. Using permanent connection for all your requests and responses is necessary due to the high performance and throughput requirements of card transaction processing.

To work with such connections, you need to know about binary framing and interleaving.

## Binary framing

Binary framing is a message framing technique where you send a fixed-length header specifying the message's length, followed by the message itself. This header allows the recipient to determine the exact length of the message before attempting to read and parse its contents.

## **Interleaving**

Interleaving is the ability to send and receive multiple requests and responses simultaneously over a single connection. Clients can send multiple requests without waiting for responses, and servers can send multiple responses without waiting for requests. This also means that responses can be received in a random order, which requires a way to match responses with requests. In the case of ISO 8583, each message (sent or received) should have a System Trace Audit Number (STAN), which is used to match requests and responses.

## Golang Packages for ISO8583

For our project we will use the following Go packages for efficient message handling and network management:

- [moov-io/iso8583](https://github.com/moov-io/iso8583) - A Golang library for packing and unpacking ISO 8583 messages, providing a seamless way to work with the message specifications.
- [moov-io/iso8583-connection](https://github.com/moov-io/iso8583-connection) - A Golang library that streamlines network management for ISO 8583, handling crucial tasks such as connection establishment, TLS configuration, heartbeat/echo/idle messaging, and request processing. Additionally, it features a connection pool with a re-connect capability to enhance reliability and performance.

# Implementing Acquiring and Issuing with ISO8583 and Golang

## Setup

I don’t want to spend a lot time here :D

## To the Work

Our project will consist of two web applications: the issuer and the acquirer. We will go step-by-step, implementing all necessary functionality for each side as required. It's a demo project, so many aspects important for a production system are ***out of scope***. Our flow will be driven by the following scenario, which closely resembles real-life situations:

1. Create an account with a `$100` balance in the Issuer
2. Issue a card for the created account
3. Create a new merchant for the Acquirer
4. Process a payment request for the merchant using the issued card
5. Verify that the payment is authorized in the Acquirer
6. Verify that an authorized transaction exists in the Issuer for the card
7. Check the transaction details to ensure they match the payment information
8. Check the merchant details of the transaction to ensure they match the merchant information
9. Verify that the account's available balance and hold balance have been updated accordingly

## End-to-end test

Let’s write an end-to-end test that will drive our development:

```go
func TestEndToEndTransaction(t *testing.T) {
	// Given
	// Create an account with $100 balance
	// Issue a card for the account
	// Create a new merchant for the acquirer
	
	// When Acquirer receives the payment request
	// for the merchant with the issued card
	
	// Then There should be an authorized transaction
	// in the acquirer

	// And There should be an authorized transaction
	// for the card in the issuer

	// And Account's available balance
	// should be less by the transaction amount
}
```

Implementing all steps of this test is the goal of our project. If test passes - we are done!

## TBC

...
