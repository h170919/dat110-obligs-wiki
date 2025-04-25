# Chapter 5: RPC Data Marshalling/Unmarshalling

In the [previous chapter](04_rpc_server_.md), we saw how the **RPC Server** acts as the central receptionist on the server side, receiving requests and dispatching them to the correct specialist (like `SensorImpl`). We noticed that both the [RPC Client](03_rpc_client_.md) and [RPC Server](04_rpc_server_.md), as well as the Stubs and Skeletons, frequently use helper functions from `RPCUtils` like `marshallInteger`, `unmarshallString`, `encapsulate`, and `decapsulate`.

But what do these functions actually *do*? When the `SensorStub` wants to send a request to read the temperature, it doesn't need any parameters. It calls `RPCUtils.marshallVoid()`. When it gets the response, it calls `RPCUtils.unmarshallInteger()` to get the temperature value. How does Java data like an `int` or even "nothing" (void) get turned into bytes that can be sent over the network, and how are bytes turned back into useful Java data?

This process of converting data between its friendly Java form and a network-friendly byte form is called **Marshalling** and **Unmarshalling**.

## What is Marshalling/Unmarshalling? It's Like Packing Boxes!

Imagine you want to send a fragile vase (your Java data, like an `int` or a `String`) through the mail (the network). You can't just slap a stamp on the vase and hope for the best!

1.  **Marshalling (Packing):** You need to carefully wrap the vase in bubble wrap and put it into a standard-sized cardboard box (a `byte[]` array). This process of converting your specific item (Java data) into a standardized package (byte array) suitable for shipping is **marshalling**.
2.  **Unmarshalling (Unpacking):** When the box arrives at its destination, the recipient opens the box and carefully unwraps the bubble wrap to get the original vase back. This process of converting the standardized package (byte array) back into the specific item (Java data) is **unmarshalling**.

So, in our RPC world:
*   **Marshalling:** Converts Java data types (like `int`, `String`, `boolean`, or even `void`) into a sequence of bytes (`byte[]`).
*   **Unmarshalling:** Converts a sequence of bytes (`byte[]`) back into the original Java data type.

This is essential because networks primarily understand how to send and receive streams of raw bytes, not complex Java objects directly.

## Marshalling/Unmarshalling Specific Data Types (`RPCUtils`)

The `RPCUtils.java` class provides helper functions to pack (marshall) and unpack (unmarshall) the specific data types we support in our simple RPC system: `int`, `String`, `boolean`, and `void`.

Let's look at a couple of examples:

**Integers:**

An integer (`int`) in Java usually takes up 4 bytes of memory. Marshalling converts this 4-byte representation into a `byte[]` array of length 4. Unmarshalling does the reverse.

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Simplified Integer Marshalling)

import java.nio.ByteBuffer; // Used for easy conversion

public class RPCUtils {

    // integer to byte array representation
    public static byte[] marshallInteger(int x) {
        // Create a buffer that can hold exactly 4 bytes (size of an int)
        ByteBuffer buffer = ByteBuffer.allocate(Integer.BYTES);
        // Put the integer into the buffer
        buffer.putInt(x);
        // Get the underlying byte array from the buffer
        byte[] encoded = buffer.array();
        return encoded;
    }

    // byte array representation to integer
    public static int unmarshallInteger(byte[] data) {
        // Wrap the incoming byte array into a buffer
        ByteBuffer buffer = ByteBuffer.wrap(data);
        // Read an integer from the buffer
        int decoded = buffer.getInt();
        return decoded;
    }
    // ... other marshall/unmarshall methods ...
}
```
*   `marshallInteger`: Uses Java's `ByteBuffer` to easily convert the `int` into a 4-byte array.
*   `unmarshallInteger`: Uses `ByteBuffer` again to read 4 bytes from the input `data` array and interpret them as an `int`.

**Strings:**

Strings are sequences of characters. Marshalling converts the string into a byte array using a standard character encoding (like UTF-8). Unmarshalling converts the byte array back into a string.

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Simplified String Marshalling)

public class RPCUtils {
    // ... other methods ...

    // convert String to byte array
    public static byte[] marshallString(String str) {
        // Use the built-in String method to get bytes (usually UTF-8)
        byte[] encoded = str.getBytes();
        return encoded;
    }

    // convert byte array to a String
    public static String unmarshallString(byte[] data) {
        // Use the String constructor that takes bytes
        String decoded = new String(data);
        return decoded;
    }
    // ... other methods ...
}
```
*   `marshallString`: Simply uses the `getBytes()` method available on all Java strings.
*   `unmarshallString`: Uses the `String` constructor that accepts a byte array.

**Void (Nothing):**

Sometimes, a remote method doesn't need any parameters (like `read()`) or doesn't return anything (like `write()`). How do we marshall "nothing"? We represent it as an empty byte array (`new byte[0]`).

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Simplified Void Marshalling)

public class RPCUtils {
    // ... other methods ...

    public static byte[] marshallVoid() {
        // Return an array with zero bytes
        return new byte[0];
    }

    public static void unmarshallVoid(byte[] data) {
        // Just check that the received data is indeed empty
        if (data.length != 0) {
            throw new IllegalArgumentException("Expected void (empty data), but received data.");
        }
        // No value to return!
    }
    // ... other methods ...
}
```
*   `marshallVoid`: Creates and returns a zero-length byte array.
*   `unmarshallVoid`: Doesn't return anything, but it checks if the received byte array is empty, throwing an error if it's not.

**Booleans:**

A boolean (`true` or `false`) can be represented by a single byte. For example, `1` for `true` and `0` for `false`.

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Boolean Marshalling)

public class RPCUtils {
    // ... other methods ...

    // convert boolean to a byte array representation
    public static byte[] marshallBoolean(boolean b) {
        byte[] encoded = new byte[1];
        encoded[0] = (b ? (byte)1 : (byte)0); // 1 if true, 0 if false
        return encoded;
    }

    // convert byte array to a boolean representation
    public static boolean unmarshallBoolean(byte[] data) {
        // True if the byte is not 0
        return (data[0] != 0);
    }
    // ... other methods ...
}
```

These marshalling/unmarshalling functions are used by the [RPC Client Stub](01_rpc_client_stub_.md) and the [RPC Server Implementation (Skeleton)](02_rpc_server_implementation__skeleton_.md) to prepare parameters and interpret return values.

## RPC Message Structure: Encapsulation/Decapsulation

Okay, so we know how to pack the "vase" (our data like an `int`) into a box (`byte[]`). But when we send this box over the network, the receiver needs to know *what this box is for*. Is it the result of a `read` call? Is it the parameter for a `write` call?

The raw marshalled data (like the 4 bytes for an integer) isn't enough. We need to add a label to the box: the **RPC ID**. This ID tells the server which remote method the message relates to (e.g., `Common.READ_RPCID` or `Common.WRITE_RPCID`).

**Encapsulation (Adding the Label):** Before sending the message, the [RPC Client](03_rpc_client_.md) (or the [RPC Server](04_rpc_server_.md) sending a reply) needs to combine the RPC ID and the marshalled data (payload) into a single byte array. This is **encapsulation**. Our format is simple: the first byte is the RPC ID, and the rest of the bytes are the marshalled payload.

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Simplified Encapsulation)

public class RPCUtils {
    // ... marshall/unmarshall methods ...

    public static byte[] encapsulate(byte rpcid, byte[] payload) {
        // Create a new array large enough for the ID (1 byte) + payload
        byte[] rpcmsg = new byte[1 + payload.length];

        // Put the RPC ID in the first position
        rpcmsg[0] = rpcid;

        // Copy the payload bytes into the rest of the array
        System.arraycopy(payload, 0, rpcmsg, 1, payload.length);

        return rpcmsg;
    }
    // ... decapsulate method ...
}
```
*   `encapsulate`: Creates a new byte array one byte larger than the `payload`. It puts the `rpcid` at the beginning and copies the `payload` bytes immediately after it.

**Decapsulation (Reading the Label and Opening the Box):** When the [RPC Server](04_rpc_server_.md) (or [RPC Client](03_rpc_client_.md) receiving a reply) receives a message, it needs to perform the reverse: extract the RPC ID and the marshalled payload. This is **decapsulation**.

```java
// File: src/main/java/no/hvl/dat110/rpc/RPCUtils.java (Simplified Decapsulation)

import java.util.Arrays; // Used for easy array copying

public class RPCUtils {
    // ... marshall/unmarshall/encapsulate methods ...

    public static byte[] decapsulate(byte[] rpcmsg) {
        // Check if the message is valid (must have at least the ID byte)
        if (rpcmsg == null || rpcmsg.length < 1) {
            throw new IllegalArgumentException("Invalid RPC message for decapsulation");
        }

        // The payload is everything *except* the first byte (the RPC ID)
        byte[] payload = Arrays.copyOfRange(rpcmsg, 1, rpcmsg.length);

        return payload;
        // Note: The caller needs to get the RPC ID separately (rpcmsg[0])
        // before calling decapsulate.
    }
}
```
*   `decapsulate`: Takes the full received message (`rpcmsg`). It extracts and returns a new byte array containing everything *from the second byte onwards* (the payload). The code that *calls* `decapsulate` (like in `RPCServer.run()` or `RPCClient.call()`) is responsible for looking at the first byte (`rpcmsg[0]`) separately to get the RPC ID.

Here's what the encapsulated message looks like:

```
[ RPC ID (1 byte) | Marshalled Parameter/Result Bytes (Payload) ]
```

## Putting It All Together: The Data Flow

Let's trace how data is transformed during a `sensor.read()` call:

```mermaid
sequenceDiagram
    participant Stub as SensorStub
    participant RPCClient
    participant Network
    participant RPCServer
    participant Skeleton as SensorImpl

    Note over Stub: Wants to call read(). No parameters.
    Stub->>Stub: Calls RPCUtils.marshallVoid() -> gets empty byte[] (param_bytes)
    Stub->>RPCClient: call(READ_RPCID, param_bytes)
    Note over RPCClient: Needs to send message.
    RPCClient->>RPCClient: Calls RPCUtils.encapsulate(READ_RPCID, param_bytes) -> gets [READ_RPCID] (msg_bytes)
    RPCClient->>Network: Sends msg_bytes
    Network->>RPCServer: Receives msg_bytes ([READ_RPCID])
    Note over RPCServer: Needs to process message.
    RPCServer->>RPCServer: Extracts ID = msg_bytes[0] (READ_RPCID)
    RPCServer->>RPCServer: Calls RPCUtils.decapsulate(msg_bytes) -> gets empty byte[] (param_bytes')
    RPCServer->>Skeleton: invoke(param_bytes')
    Note over Skeleton: Needs parameters.
    Skeleton->>Skeleton: Calls RPCUtils.unmarshallVoid(param_bytes') -> Checks it's empty. OK.
    Skeleton->>Skeleton: Calls internal read() -> gets temperature (e.g., 17)
    Note over Skeleton: Needs to return result.
    Skeleton->>Skeleton: Calls RPCUtils.marshallInteger(17) -> gets 4 bytes for 17 (result_bytes)
    Skeleton-->>RPCServer: Returns result_bytes
    Note over RPCServer: Needs to send reply.
    RPCServer->>RPCServer: Calls RPCUtils.encapsulate(READ_RPCID, result_bytes) -> gets [READ_RPCID | 4 bytes for 17] (reply_bytes)
    RPCServer->>Network: Sends reply_bytes
    Network->>RPCClient: Receives reply_bytes
    Note over RPCClient: Needs to process reply.
    RPCClient->>RPCClient: Calls RPCUtils.decapsulate(reply_bytes) -> gets 4 bytes for 17 (result_bytes')
    RPCClient-->>Stub: Returns result_bytes'
    Note over Stub: Needs result value.
    Stub->>Stub: Calls RPCUtils.unmarshallInteger(result_bytes') -> gets 17
    Stub-->>Controller: Returns 17
end
```

This diagram shows the journey:
1.  Stub **marshalls** parameters.
2.  RPCClient **encapsulates** the ID and marshalled parameters.
3.  RPCServer **decapsulates** the received message to get the ID and marshalled parameters.
4.  Skeleton **unmarshalls** the parameters to use them.
5.  Skeleton **marshalls** the result.
6.  RPCServer **encapsulates** the ID and marshalled result for the reply.
7.  RPCClient **decapsulates** the reply to get the marshalled result.
8.  Stub **unmarshalls** the result to return it to the caller.

## Conclusion

Marshalling/Unmarshalling is the crucial process of converting Java data into byte arrays for network travel and back again, like packing and unpacking a box. Encapsulation/Decapsulation is the process of adding (and removing) the necessary RPC ID label to this box so the receiver knows what the message is for.

The `RPCUtils` class provides the tools for both:
*   `marshallX`/`unmarshallX` methods for packing/unpacking specific data types (used by Stubs and Skeletons).
*   `encapsulate`/`decapsulate` methods for adding/removing the RPC ID label to the packed data (used by RPC Client and RPC Server).

Without these steps, our RPC components wouldn't be able to exchange meaningful information over the network.

Now that we understand how data is prepared for sending, how is it *actually* sent and received as a reliable message? That's the job of the layer below RPC: the Messaging Layer.

Next up: [Messaging Connection](06_messaging_connection_.md)

---
