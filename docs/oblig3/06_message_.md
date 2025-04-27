# Chapter 6: Message

Welcome to Chapter 6! In the [previous chapter](05_chord_protocol__ring__lookup__stabilization_.md), we learned how the **Chord Protocol** acts like a clever directory and navigation system, allowing nodes to organize themselves into a ring and efficiently find the node responsible for any given piece of data using its ID.

Now, imagine Node A has used Chord to find that Node B is responsible for storing a replica of "my_notes.txt". Node A needs to send a request to Node B using [Remote Communication (RMI)](03_remote_communication__rmi__.md). But what exactly should Node A send? It needs to include the file name, the replica ID, the actual file data, maybe who sent the request, and perhaps some special instructions. Just sending raw bits of data isn't organized. We need a standard format for these communications.

This chapter introduces the **Message** concept. It's a standardized data structure, like a pre-printed form or memo, used to package information neatly when nodes exchange data or requests.

**What You'll Learn:**

*   Why a standardized `Message` format is useful for communication.
*   What key pieces of information are included in a `Message`.
*   How `Message` objects are created and used in the system.
*   How `Message` objects are also used to store file metadata on [Nodes (Peers)](02_node__peer__.md).

## The Standardized Memo Analogy

Think of inter-office communication in a large company. If everyone just scribbled notes on random pieces of paper, it would be chaos! Important details might be missed, and processing requests would be slow. Instead, companies often use standardized memo forms.

A typical memo form might have fields for:

*   **FROM:** (Sender's Name/Department)
*   **TO:** (Recipient's Name/Department - *Implied by delivery*)
*   **DATE:** (When it was sent)
*   **SUBJECT:** (What is this memo about?)
*   **BODY:** (The actual content or request)
*   **ATTACHMENTS:** (Any accompanying documents)
*   **URGENT?** (Checkbox for priority)
*   **ACTION REQUIRED?** (Checkbox for status)

Our `Message` class is exactly like this standardized memo for our distributed system nodes. It ensures that whenever nodes communicate or store information about files, all the necessary details are present in a predictable format.

## Key Information in a `Message`

Let's look at the important fields inside our `Message` "memo":

1.  **Sender Information:**
    *   `nodeID` ([BigInteger](04_hashing___id_space_.md)): The unique ID of the node sending or associated with this message. (Like the sender's official employee ID).
    *   `nodename` (String): The human-readable name of the sender node (e.g., "process1").
    *   `port` (int): The communication port of the sender node.

2.  **File Information (if applicable):**
    *   `nameOfFile` (String): The original name of the file this message relates to (e.g., "my\_notes"). (Like the "SUBJECT" line).
    *   `hashOfFile` ([BigInteger](04_hashing___id_space_.md)): The specific ID of the file replica this message is about (e.g., the hash of "my\_notes1").
    *   `bytesOfFile` (`byte[]`): The actual content of the file replica. (Like the "ATTACHMENTS"). This isn't always filled, only when sending the file data itself.
    *   `filepath` (String): Sometimes used to store the original path of the file, though less critical for internal node communication.

3.  **Ordering & Status:**
    *   `clock` (int): A Lamport clock timestamp. This is a simple counter used to help determine the order of events across different nodes, which is important for things like [Consistency (Remote-Write Protocol)](07_consistency__remote_write_protocol_.md). (Like a sequence number on the memo).
    *   `acknowledged` (boolean): A flag to indicate if this message has been acknowledged or processed. (Like a "RECEIVED" stamp).
    *   `primaryServer` (boolean): A special flag indicating if the node holding this file replica is the "primary" copy. This is crucial for coordinating updates, as we'll see in the [Consistency (Remote-Write Protocol)](07_consistency__remote_write_protocol_.md) chapter. (Like an "ACTION REQUIRED BY:" checkbox).

## Using `Message` Objects

`Message` objects are used in two primary ways:

**1. Packaging Information for RMI Calls:**

When one node needs to send data to another using [Remote Communication (RMI)](03_remote_communication__rmi__.md), it often packages the necessary information into a `Message` object.

**Example:** Node A (running `FileManager`) wants to tell Node B to save a file replica.

```java
// --- On Node A (inside FileManager or similar) ---
NodeInterface nodeB_stub = Util.getProcessStub("process2", 9092); // Get RMI stub for Node B

// Details of the replica to be saved
String filename = "my_notes";
BigInteger replicaId = Hash.hashOf("my_notes1");
byte[] fileContent = //... read file content ...
boolean isPrimary = false; // Assume this is not the primary replica

// Create a Message object to package the info
Message fileInfoMsg = new Message(nodeA.getNodeID(), nodeA.getNodeName(), nodeA.getPort());
fileInfoMsg.setNameOfFile(filename);
fileInfoMsg.setHashOfFile(replicaId);
fileInfoMsg.setBytesOfFile(fileContent);
fileInfoMsg.setPrimaryServer(isPrimary);
// fileInfoMsg.setClock(...); // Lamport clock would be set here too

try {
    // Send the entire Message object via RMI
    // (Note: In the actual project, RMI methods might take individual parameters
    // derived from the Message, or the Message itself might be adapted/wrapped)

    // Example hypothetical RMI call:
    // nodeB_stub.receiveAndStoreFile(fileInfoMsg);

    // OR, more likely based on current NodeInterface:
    nodeB_stub.saveFileContent(fileInfoMsg.getNameOfFile(),
                               fileInfoMsg.getHashOfFile(),
                               fileInfoMsg.getBytesOfFile(),
                               fileInfoMsg.isPrimaryServer());

    System.out.println("Sent file info for " + filename + " to Node B.");

} catch (RemoteException e) {
    System.err.println("Error sending file info: " + e.getMessage());
}
```

*Explanation:* Node A gathers all the necessary details about the file replica (`filename`, `replicaId`, `fileContent`, `isPrimary`, plus sender info). It creates a `Message` object and uses setter methods (like `setNameOfFile`, `setBytesOfFile`) to populate it. This organized `Message` (or the information extracted from it) is then sent over the network using RMI to Node B.

**2. Storing File Metadata on Nodes:**

As mentioned in the [Node (Peer)](02_node__peer__.md) chapter, each node maintains a `filesMetadata` map:

```java
// Inside Node.java
private Map<BigInteger, Message> filesMetadata; // Stores info about files this node holds
```

This map uses the file replica ID (`BigInteger`) as the key. The *value* stored for each key is a `Message` object! This `Message` object holds all the relevant metadata about that specific file replica stored on *this* node (like its original name, its replica ID, whether it's the primary, its Lamport clock value, etc.). It typically *doesn't* store the `bytesOfFile` in this map to save memory; the actual file content might be stored separately on disk, referenced by the metadata.

**Example:** When Node B receives the `saveFileContent` call from Node A.

```java
// --- On Node B (inside Node.java's saveFileContent method) ---
@Override
public void saveFileContent(String filename, BigInteger fileID, byte[] bytesOfFile, boolean primary)
        throws RemoteException {

    System.out.println("Node " + nodename + ": Received request to save " + filename + " (ID: " + fileID + ")");

    // Create a Message object to store the *metadata* for this replica
    // Use info about *this* node (Node B) as the 'sender' in the metadata context
    Message metaData = new Message(this.nodeID, this.nodename, this.port);
    metaData.setNameOfFile(filename);
    metaData.setHashOfFile(fileID);
    // metaData.setBytesOfFile(null); // Don't store bytes in the metadata map
    metaData.setPrimaryServer(primary);
    // metaData.setClock(0); // Initialize clock

    // Store the metadata Message in the map, keyed by the replica ID
    filesMetadata.put(fileID, metaData);

    // Store the actual file content (bytesOfFile) somewhere else, e.g., on disk
    // storeBytesToFileSystem(fileID, bytesOfFile); // Hypothetical method

    System.out.println("Node " + nodename + ": Stored metadata for " + filename);
}
```

*Explanation:* When Node B is asked to save a file replica, it creates a *new* `Message` object to represent the *metadata* it needs to remember about this replica. It populates this `Message` with the provided details (`filename`, `fileID`, `primary` status) and information about itself (`nodeID`, `nodename`, `port`). This metadata `Message` is then stored in the `filesMetadata` map. This allows Node B to later answer questions like "What files are you storing?" or "Are you the primary for file replica X?".

## Under the Hood: The `Message.java` Class

The `Message` class itself is relatively simple. It's primarily a container for data.

```java
// File: src/main/java/no/hvl/dat110/middleware/Message.java (Simplified)

package no.hvl.dat110.middleware;

import java.io.Serializable; // Important!
import java.math.BigInteger;
import java.rmi.RemoteException;

// Message must implement Serializable to be sent via RMI
public class Message implements Serializable {

    // Unique ID for serialization compatibility
	private static final long serialVersionUID = 1L;

	// Fields we discussed:
	private int clock = 0;             // Lamport clock
	private BigInteger nodeID;         // Sender/Owner Node ID
	private String nodename;           // Sender/Owner Node Name
	private int port;                  // Sender/Owner Node Port
	private boolean acknowledged = false; // Acknowledgment status
	private String filepath;           // Original file path (optional)

	private byte[] bytesOfFile;        // File content (used for transfer)
	private BigInteger hashOfFile;     // File replica ID
	private String nameOfFile;         // Original file name

	private boolean primaryServer;     // Is this the primary replica?


	// Constructor to set basic node info
	public Message(BigInteger nodeID, String nodename, int port) {
		this.nodeID = nodeID;
		this.nodename = nodename;
		this.port = port;
	}

	// --- Getters and Setters for all fields ---
    // Example:
    public String getNameOfFile() {
		return nameOfFile;
	}
	public void setNameOfFile(String nameOfFile) {
		this.nameOfFile = nameOfFile;
	}
    // ... other getters and setters ...
}
```

*Explanation:*
*   **`implements Serializable`:** This is crucial. It's a marker interface from Java that tells the Java runtime environment that objects of this class are allowed to be "flattened" into a sequence of bytes (serialized) and sent over a network stream (like RMI does) or saved to a file. The receiving end can then reconstruct the original object from these bytes (deserialize). Without `Serializable`, RMI wouldn't be able to transmit `Message` objects.
*   **`serialVersionUID`:** This is a version number used during serialization/deserialization to ensure the sender and receiver are using compatible versions of the `Message` class.
*   **Fields:** It contains private fields for all the pieces of information we discussed (sender, file, status).
*   **Constructor:** A simple constructor to initialize the basic node information.
*   **Getters and Setters:** Standard methods (`getFieldName()`, `setFieldName(value)`) are provided to access and modify the private fields, following standard Java practice for data classes (JavaBeans).

There's no complex logic inside the `Message` class itself; it's just a well-defined container, our standardized memo format.

## Conclusion

In this chapter, we learned about the `Message` class, which acts as a standardized information package – like a memo – for our distributed system. It bundles together crucial details about the sender, the file involved, status flags, and ordering information.

We saw how `Message` objects are essential for packaging data sent between nodes using [Remote Communication (RMI)](03_remote_communication__rmi_.md) and how they are also used by each [Node (Peer)](02_node__peer__.md) to store the metadata (`filesMetadata`) about the file replicas it manages. Using this standard format makes communication and data management much more organized and reliable.

Now that we have nodes organized ([Chord Protocol (Ring, Lookup, Stabilization)](05_chord_protocol__ring__lookup__stabilization_.md)) and a standard way to package information (`Message`), a critical question arises: if we have multiple copies (replicas) of a file stored on different nodes, how do we ensure they all stay consistent when the file is updated? We'll tackle this challenge in the next chapter on [Consistency (Remote-Write Protocol)](07_consistency__remote_write_protocol_.md).

---
