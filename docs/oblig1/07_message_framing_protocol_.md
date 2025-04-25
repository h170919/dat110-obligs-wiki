# Chapter 7: Message Framing Protocol

Welcome to the final chapter! In the [previous chapter](06_messaging_connection_.md), we explored the **Messaging Connection** layer. We saw how it acts like a reliable postal service, providing `send` and `receive` methods to exchange `Message` objects between a client and server. We briefly mentioned that there's some "magic" happening inside `send` and `receive` to make sure whole messages are sent and received correctly, even though the underlying network connection (TCP) just sees a continuous stream of bytes.

It's time to reveal that magic! It's called the **Message Framing Protocol**.

## The Problem: Just a Stream of Bytes

Imagine you're reading a very long scroll of text that has no punctuation or spaces between words. It would be incredibly difficult to figure out where one word ends and the next begins, let alone understand the sentences!

TCP network connections are a bit like that scroll. They provide a reliable *stream* of bytes from the sender to the receiver. If the sender sends "HELLO" (5 bytes) and then immediately sends "WORLD" (5 bytes), the receiver might just get a stream like "HELLOWORLD" (10 bytes). The receiver doesn't automatically know that this was originally two separate messages.

Our [Messaging Connection](06_messaging_connection_.md) needs to send and receive distinct `Message` objects. If the [RPC Client](03_rpc_client_.md) sends a request message, and then maybe another one later, the [RPC Server](04_rpc_server_.md) needs to be able to receive the *first* message completely, process it, and *then* receive the *second* message completely. How can we add structure to this byte stream?

## The Solution: Message Framing Protocol - Standard Envelopes!

The solution is to agree on a set of rules – a **protocol** – for how messages are packaged, or **framed**, before being sent over the stream. This ensures the receiver can identify the boundaries of each message.

Think of it like agreeing to always send letters using standard-sized envelopes. Even if you put multiple envelopes into the mailbag one after another, the recipient can easily pick out one envelope, open it, read the letter inside, and then pick out the next envelope.

Our **Message Framing Protocol** defines the "standard envelope" for our system.

**Our Specific Protocol:**

In this project (`dat110-project1-gruppe69`), we use a simple framing protocol:

1.  **Fixed-Size Segments:** All data is sent in fixed-size chunks called **segments**. Each segment is exactly **128 bytes** long.
2.  **Length Header:** The very **first byte** of each 128-byte segment is special. It acts as a **header** and tells the receiver the **length** of the actual payload data contained within *that segment*. This length can be anywhere from 0 to 127 bytes.
3.  **Payload:** The actual message data (the bytes from the `Message` object) follows the length byte.
4.  **Padding:** Since the segment *must* be 128 bytes, but the payload might be shorter (e.g., only 10 bytes), the remaining bytes in the segment after the payload are just **padding**. They are sent but ignored by the receiver.

Here's what our 128-byte "envelope" (segment) looks like:

```mermaid
graph LR
    A[Segment (128 bytes total)] --> B(Byte 1: Length L);
    A --> C(Bytes 2 to L+1: Payload Data);
    A --> D(Bytes L+2 to 128: Padding);
    subgraph Fixed Size Frame
        direction LR
        B --- C --- D;
    end
    style A fill:#f9f,stroke:#333,stroke-width:2px
```

*   The first byte tells us `L`, the length of the *real* data.
*   The next `L` bytes are the actual payload we care about.
*   The rest of the 128 bytes are just filler (padding).

## Implementing the Framing: `MessageUtils`

The rules of our protocol are implemented in the helper methods of the `MessageUtils.java` class, specifically `encapsulate` and `decapsulate`. These are the methods used internally by `MessageConnection`'s `send` and `receive`.

**1. Packing the Envelope (`encapsulate`)**

When `MessageConnection.send(message)` is called, it needs to take the `message`'s data and put it into our standard 128-byte envelope format. It uses `MessageUtils.encapsulate` for this.

```java
// File: src/main/java/no/hvl/dat110/messaging/MessageUtils.java (Encapsulate)

public class MessageUtils {

    public static final int SEGMENTSIZE = 128; // Our fixed envelope size

    // Takes the message payload and puts it into a 128-byte segment
    public static byte[] encapsulate(Message message) {
        byte[] data = message.getData(); // Get the raw payload bytes (0-127 bytes)
        byte[] segment = new byte[SEGMENTSIZE]; // Create the 128-byte empty envelope

        // Rule 1: Put the length of the payload in the first byte
        segment[0] = (byte) data.length;

        // Rule 2: Copy the payload data into the segment, starting from byte 1
        System.arraycopy(data, 0, segment, 1, data.length);

        // Rule 3: The rest of 'segment' is padding (already 0s)
        return segment; // Return the filled 128-byte envelope
    }

    // ... decapsulate method ...
}
```

Let's break it down:
*   It gets the payload `data` from the `Message` object. Remember from `Message.java`, this `data` is guaranteed to be 127 bytes or less.
*   It creates a brand new byte array called `segment` that is exactly 128 bytes long.
*   `segment[0] = (byte) data.length;` This crucial step writes the *length* of the payload into the very first byte of the segment.
*   `System.arraycopy(...)` copies the actual payload `data` into the `segment`, but it starts copying *at index 1* (the second byte), leaving the first byte for the length.
*   The method returns the 128-byte `segment`. This is what `MessageConnection` actually writes to the network stream.

**2. Unpacking the Envelope (`decapsulate`)**

When `MessageConnection.receive()` is called, it first reads exactly 128 bytes from the network stream (because it knows all messages arrive in 128-byte segments). It then needs to extract the *actual* payload from this 128-byte segment. It uses `MessageUtils.decapsulate` for this.

```java
// File: src/main/java/no/hvl/dat110/messaging/MessageUtils.java (Decapsulate)

import java.util.Arrays; // Used for array copying

public class MessageUtils {
    // ... SEGMENTSIZE constant and encapsulate method ...

    // Takes a 128-byte segment and extracts the original payload data
    public static Message decapsulate(byte[] segment) {

        // Basic checks for valid segment
        if (segment == null || segment.length != SEGMENTSIZE) {
           throw new IllegalArgumentException("Invalid segment received");
        }

        // Rule 1: Read the payload length from the first byte
        // (& 0xFF ensures we treat the byte as unsigned, 0-255)
        int length = segment[0] & 0xFF;

        // Sanity check: Length shouldn't exceed allowed payload size
        if (length > SEGMENTSIZE - 1) { // Max payload is 127
             throw new IllegalArgumentException("Segment length field is too large");
        }

        // Rule 2: Extract the payload data
        // Create a new array of the correct size ('length')
        byte[] data = new byte[length];
        // Copy 'length' bytes from the segment, starting from byte 1
        System.arraycopy(segment, 1, data, 0, length);

        // Rule 3: The padding is ignored!

        // Create and return a new Message containing only the extracted data
        return new Message(data);
    }
}
```

Here's how it works:
*   It receives the 128-byte `segment` that was read from the network.
*   `int length = segment[0] & 0xFF;` reads the first byte to figure out how long the *actual* payload is. (The `& 0xFF` is a technical detail to handle Java's byte representation correctly).
*   It creates a *new* byte array called `data` that is exactly `length` bytes long – just big enough for the payload.
*   `System.arraycopy(...)` copies `length` bytes *from* the received `segment` (starting at index 1) *into* the new `data` array.
*   Finally, it creates a new `Message` object using only the extracted `data` and returns it. This `Message` now contains only the original payload sent by the sender, with the framing and padding removed.

## The Whole Picture

So, the [Messaging Connection](06_messaging_connection_.md) layer uses this framing protocol like this:

1.  **Sender (`MessageConnection.send`):**
    *   Gets the `Message` object (containing payload from, say, [RPC Data Marshalling/Unmarshalling](05_rpc_data_marshalling_unmarshalling_.md)).
    *   Calls `MessageUtils.encapsulate` to wrap the payload in a 128-byte segment with the length header.
    *   Writes the entire 128-byte segment to the TCP output stream.

2.  **Receiver (`MessageConnection.receive`):**
    *   Reads exactly 128 bytes from the TCP input stream (because it expects a segment).
    *   Calls `MessageUtils.decapsulate` on the received 128-byte segment.
    *   `decapsulate` reads the length header, extracts the payload bytes, and ignores the padding.
    *   `decapsulate` returns a new `Message` object containing only the original payload.
    *   `receive` returns this payload-only `Message` object to the caller (e.g., the [RPC Server](04_rpc_server_.md)).

This ensures that even though TCP is just a stream, the `MessageConnection` layer can reliably send and receive distinct messages because the **Message Framing Protocol** provides the necessary structure.

## Conclusion

The **Message Framing Protocol** is a fundamental concept in network programming. It defines the rules for how to structure data sent over a stream-based connection so that the receiver can distinguish individual messages. In our project, we use a simple fixed-size segment protocol: every message travels inside a 128-byte "envelope," where the first byte declares the length of the actual content inside (0-127 bytes), and the rest is potential padding.

The `MessageUtils.encapsulate` and `MessageUtils.decapsulate` methods implement this protocol, allowing our [Messaging Connection](06_messaging_connection_.md) layer to provide a clean `send(Message)` and `receive()` interface to the higher RPC layers.

With this final piece, we've explored the entire stack, from the high-level convenience of RPC Stubs down to the details of packaging bytes for reliable network transmission. Congratulations on completing the tutorial for `dat110-project1-gruppe69`!

---
