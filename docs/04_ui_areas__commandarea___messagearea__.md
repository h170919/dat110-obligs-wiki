# Chapter 4: UI Areas (CommandArea & MessageArea)

In the [previous chapter](03_chapp__application_entry_point__.md), we saw how `Chapp` acts like a stage manager, setting up the main window (`Stage`) and creating the initial instances of our UI components. It brought the actors (`CommandArea`, `MessageArea`) onto the stage but didn't delve into what those actors actually *do* or how they are built.

Now, let's look closely at two of those main actors: `CommandArea` and `MessageArea`. Think about any chat application you've used. You usually have:

1.  A place where you type your messages, maybe choose a topic, and click "Send" or "Connect".
2.  A separate area where you see the messages from other people appearing.

It wouldn't make sense to mix these up, right? Having dedicated areas makes the application much easier to use. Our `CommandArea` and `MessageArea` classes define these distinct sections.

## The Dashboard Analogy

Imagine the dashboard of a car. You have one area with controls: the steering wheel, pedals, buttons for lights or wipers. This is where *you* interact with the car to tell it what to do. This is like our `CommandArea`.

Then, you have another area with displays: the speedometer, fuel gauge, warning lights. This shows you information *from* the car about its status. This is like our `MessageArea`.

`CommandArea` and `MessageArea` are Java classes that use JavaFX elements to build these visual panels for our chat application.

## `CommandArea`: The Control Panel

The `CommandArea` class (`CommandArea.java`) is responsible for creating the part of the user interface where the user gives commands or types messages. It contains all the buttons, text boxes, and labels needed for interaction.

**What's Inside?**

*   **Buttons:** For actions like "Connect", "Create Topic", "Subscribe", "Publish".
*   **Text Fields:** For typing in the topic name or the message content.
*   **Labels:** To display static text or simple information like the connected server or current username.
*   **Layout:** It uses a layout pane (like `GridPane`) to arrange these elements neatly.

**Setting up the `CommandArea`**

Remember in `Chapp.java`, we saw this line:

```java
// From Chapp.java's start() method
carea.setupCommandArea(hbox, controller, marea);
```

This calls the `setupCommandArea` method inside `CommandArea.java`. Let's look at a simplified version of what that method does:

```java
// File: src/main/java/no/hvl/dat110/chapp/CommandArea.java (Simplified)
package no.hvl.dat110.chapp;

import javafx.scene.control.Button;
import javafx.scene.control.TextField;
import javafx.scene.layout.GridPane; // For arranging elements in a grid
import javafx.scene.layout.HBox;    // The horizontal box from Chapp
// ... other imports

public class CommandArea {

	private Controller controller; // Reference to the application's brain

	public void setupCommandArea(HBox hbox, Controller controller, MessageArea marea) {
		this.controller = controller; // Store the controller for later use

		// Create a grid layout to organize controls
		GridPane grid = new GridPane();
		grid.setHgap(10); // Horizontal spacing
		grid.setVgap(10); // Vertical spacing

		// --- Example: Connect Button ---
		Button connectBtn = new Button("Connect");
		grid.add(connectBtn, 0, 0); // Add button to grid at column 0, row 0

		// --- Example: Topic Text Field ---
		TextField topicField = new TextField();
		topicField.setPromptText("Enter topic name"); // Placeholder text
		grid.add(topicField, 1, 1); // Add text field at column 1, row 1

		// --- Example: Create Topic Button ---
		Button createTopicBtn = new Button("Create Topic");
		grid.add(createTopicBtn, 0, 1); // Add button at column 0, row 1

		// ... (Many other buttons and fields are created and added here) ...

		// Finally, add the organized grid to the HBox provided by Chapp
		hbox.getChildren().add(grid);
	}
}
```

*   **`setupCommandArea`:** This method takes the main `HBox` (from `Chapp`), the central `Controller` (our application's brain), and the `MessageArea` as input.
*   **`GridPane`:** A handy JavaFX layout tool that arranges elements in rows and columns, like a spreadsheet.
*   **`new Button(...)`, `new TextField()`:** These lines create the actual visual components.
*   **`grid.add(...)`:** This places each component into a specific cell in the grid.
*   **`hbox.getChildren().add(grid)`:** The entire grid, filled with controls, is added to the `HBox` that `Chapp` created, making it visible in the main window layout.

**Making Buttons Do Things**

Creating buttons is nice, but they need to *do* something when clicked! This is where the [Controller](05_controller__ui_logic_coordinator__.md) comes in. Inside `setupCommandArea`, we also set up *event handlers*.

Think of an event handler as telling the button: "When someone clicks you, call this specific method in the `Controller`."

```java
// File: src/main/java/no/hvl/dat110/chapp/CommandArea.java (Simplified event handling)

// Inside setupCommandArea, after creating createTopicBtn and topicField

createTopicBtn.setOnAction((event) -> {
	// This code runs when createTopicBtn is clicked

	String topic = topicField.getText(); // Get text from the topic input field

	if (!topic.isEmpty()) { // Make sure the user typed something
		// Tell the controller to handle the "create topic" action
		controller.createTopic(topic);
		System.out.println("Create Topic button clicked for: " + topic);
	}
});

// Similar setOnAction calls are made for connectBtn, subscribeBtn, publishBtn, etc.
```

*   **`setOnAction(...)`:** This is a standard JavaFX way to define what happens when a button (or other control) is interacted with.
*   **`controller.createTopic(topic)`:** This is the crucial part! The `CommandArea` doesn't know *how* to create a topic itself. It just grabs the topic name from the text field and tells the `Controller` to handle the logic. The `Controller` will then likely use the `Client` to send the appropriate message ([Chapter 1](01_message_hierarchy__communication_protocol__.md)) to the server.

So, the `CommandArea` is the user's input panel, collecting information and delegating the actual work to the `Controller`.

## `MessageArea`: The Information Display

The `MessageArea` class (`MessageArea.java`) builds the other main part of our UI: the section where incoming chat messages, connection confirmations, and other notifications are shown to the user.

**What's Inside?**

*   **`TextArea`:** The primary component. This is a multi-line text box designed to display (and optionally edit) text. In our case, we'll mostly use it for *displaying* incoming messages.
*   **Labels:** Maybe a title like "Messages".
*   **Layout:** It uses a layout pane (like `VBox`) to arrange its elements.

**Setting up the `MessageArea`**

Similar to `CommandArea`, `Chapp.java` calls a setup method:

```java
// From Chapp.java's start() method
marea.setupMessageAera(hbox, controller);
```

Let's look at a simplified `setupMessageAera` method:

```java
// File: src/main/java/no/hvl/dat110/chapp/MessageArea.java (Simplified)
package no.hvl.dat110.chapp;

import javafx.scene.control.Label;
import javafx.scene.control.TextArea; // The main display area
import javafx.scene.layout.HBox;     // The horizontal box from Chapp
import javafx.scene.layout.VBox;     // Vertical layout for this area
// ... other imports

public class MessageArea {

	private Controller controller;
    private TextArea messages; // The field where messages will appear

	public void setupMessageAera(HBox hbox, Controller controller) {
		this.controller = controller;

		// Create a vertical layout for this panel
		VBox vbox = new VBox();
		vbox.setPadding(new Insets(10, 10, 10, 10)); // Add some spacing

		// --- A Title Label ---
		Label msgLabel = new Label("Messages");

		// --- The Main Message Display Area ---
		messages = new TextArea(); // Create the text area
		messages.setPrefHeight(500); // Suggest a height
		messages.setPrefWidth(300);  // Suggest a width
		messages.setEditable(false); // User shouldn't type directly here!

		// Add the label and the text area to the vertical box
		vbox.getChildren().addAll(msgLabel, messages);

		// Add this vertical box to the main HBox from Chapp
		hbox.getChildren().add(vbox);
	}

	// ... (methods to start/stop message handling are also here) ...
}
```

*   **`setupMessageAera`:** Takes the main `HBox` and the `Controller`.
*   **`VBox`:** A vertical layout pane. We put the "Messages" label above the `TextArea`.
*   **`new TextArea()`:** Creates the multi-line text display area.
*   **`messages.setEditable(false)`:** This is important! It prevents the user from typing directly into the message display area. Only our application logic should add text here.
*   **`vbox.getChildren().addAll(...)`:** Adds the label and the text area to our vertical layout.
*   **`hbox.getChildren().add(vbox)`:** Adds the `MessageArea`'s layout next to the `CommandArea`'s layout within the main window's `HBox`.

**Receiving and Displaying Messages**

How does text actually appear in the `messages` TextArea? The `MessageArea` itself doesn't directly listen for network messages. That's the job of the [Client](02_client__network_interaction_logic__.md) and the [Controller](05_controller__ui_logic_coordinator__.md).

The `MessageArea` has a helper mechanism (involving `MessageHandler.java`, which we won't detail here) that periodically asks the `Controller` if any new messages have arrived. If the `Controller` provides a message, the `MessageHandler` updates the `TextArea`.

A crucial part of this is starting the message handler:

```java
// File: src/main/java/no/hvl/dat110/chapp/MessageArea.java

// Reference to the message handler helper task
private MessageHandler msghandler;
// The TextArea defined in setupMessageAera
private TextArea messages;

public void startMessageHandler () {
    // Create the handler, giving it the controller (to ask for messages)
    // and the messages TextArea (to update it)
    msghandler = new MessageHandler(controller, messages);

    // Start the handler running in the background
    msghandler.start();
}

public void stopMessageHandler () {
	// Method to stop the background task gracefully
	if (msghandler != null) {
		msghandler.doStop();
	}
}

// --- Inside MessageHandler.java (Conceptual) ---
// public void doProcess() {
//     String message = controller.receive(); // Ask controller for new message
//     if (message != null) {
//         messages.appendText(message + "\n-\n"); // Add to TextArea!
//     }
//     // Wait a bit before checking again
// }
```

*   **`startMessageHandler()`:** This method (called at the right time, often after connecting) creates and starts the `MessageHandler`.
*   **`MessageHandler`:** This helper continuously checks with the `controller.receive()` method.
*   **`messages.appendText(...)`:** If `controller.receive()` returns a message string, the handler adds it to the end of the text already in the `TextArea`.

So, the `MessageArea` primarily provides the *place* (`TextArea`) for messages to be displayed, and relies on a helper mechanism coordinated by the `Controller` to actually *get* and *display* those messages.

## How They Fit Together

```mermaid
graph TD
    subgraph User Interface Window
        CA[CommandArea (GridPane)]
        MA[MessageArea (VBox)]
    end

    User --> CA(Clicks Button, Types Text)
    CA --> CTRL(Controller)
    CTRL --> Client(Client: Sends Message)
    Client --> Server

    Server --> Client(Client: Receives Message)
    Client --> CTRL(Controller: Processes Message)
    CTRL --> MH(MessageHandler)
    MH --> MA(Updates TextArea)

    style CA fill:#ccf,stroke:#333,stroke-width:2px
    style MA fill:#cdf,stroke:#333,stroke-width:2px
```

This diagram shows the flow:
1.  The User interacts with the `CommandArea`.
2.  `CommandArea` tells the `Controller` what the user did.
3.  The `Controller` uses the `Client` to talk to the Server.
4.  When messages come back from the Server via the `Client`, the `Controller` processes them.
5.  The `Controller` makes the message available to the `MessageHandler` (part of `MessageArea`'s machinery).
6.  The `MessageHandler` updates the `TextArea` inside the `MessageArea` for the user to see.

## Conclusion

We've learned that `CommandArea` and `MessageArea` are crucial for organizing our chat application's user interface into logical sections:

*   **`CommandArea`**: Acts as the **control panel**, providing buttons and text fields for user input. It delegates actions to the `Controller`.
*   **`MessageArea`**: Acts as the **display panel**, primarily using a `TextArea` to show incoming messages received via the `Controller` and its helper `MessageHandler`.

These classes use standard JavaFX components (`Button`, `TextField`, `TextArea`, `GridPane`, `VBox`) to build their respective parts of the UI. They are created by `Chapp` and rely heavily on the `Controller` to link user actions and incoming data to the correct behavior.

Now that we understand the input and output areas of our UI, it's time to look at the central coordinator that makes them work together. In the next chapter, we'll explore the [Controller (UI Logic Coordinator)](05_controller__ui_logic_coordinator__.md), the brain that connects the user's actions in the `CommandArea` to the underlying `Client` logic and updates the `MessageArea`.

---

