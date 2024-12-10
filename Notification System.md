# Notification System in Node.js with Redis

In chat applications, when a user sends messages to another user, but the receiver is busy (or not actively engaged with the sender), the system should:

- Keep track of the number of unread messages (notifications) for each user.
- Display a notification badge next to the sender’s name or profile to indicate how many unread messages they have.
- Notify the receiver when new messages are sent by other users, even if they’re not actively looking at the chat.
- Allow users to "mark as read" when they visit the chat interface, which will remove those notifications.

This type of system is very common in **real-time messaging apps**. It ensures that the users are always aware of the messages they haven’t read yet, and the system can give an alert or badge based on unread messages.

To implement this functionality, we need the following key components:

1. **Backend** (Node.js with Redis, MySQL/MSSQL):

   - Store messages in the database.
   - Keep track of unread messages for each user (stored in the database).
   - Use Redis and WebSocket to push notifications to the client in real-time.
   - When the user accesses the chat, mark messages as read and update the notification counts.

2. **Frontend** (React.js):
   - Use WebSocket (via `react-use-websocket`) to receive real-time notifications for new messages.
   - Show the unread message count next to the sender or in a notification icon.
   - Allow users to mark messages as read and reset the notification badge count.

### Full Code and Explanation:

#### 1. **Backend: Node.js, Redis, MySQL**

##### `server.js` (Node.js)

```javascript
const express = require("express");
const mysql = require("mysql"); // Use 'mssql' for SQL Server
const redis = require("redis");
const http = require("http");
const socketIo = require("socket.io");
const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// MySQL/MSSQL client configuration
const db = mysql.createConnection({
  host: "localhost",
  user: "root",
  password: "password",
  database: "chat_app",
});

db.connect((err) => {
  if (err) throw err;
  console.log("Connected to MySQL database");
});

// Redis client configuration (for pub/sub)
const redisClient = redis.createClient();
const redisPublisher = redis.createClient();
const redisSubscriber = redis.createClient();

redisClient.on("error", (err) => {
  console.error("Redis error: ", err);
});

// Cache helper functions for unread message counts
function getUnreadMessages(userId, callback) {
  redisClient.get(`unread:${userId}`, (err, count) => {
    if (err) throw err;
    callback(count || 0); // Return 0 if no unread messages found
  });
}

function incrementUnreadMessages(userId, callback) {
  redisClient.incr(`unread:${userId}`, (err, newCount) => {
    if (err) throw err;
    callback(newCount);
  });
}

function resetUnreadMessages(userId, callback) {
  redisClient.del(`unread:${userId}`, (err) => {
    if (err) throw err;
    callback(0);
  });
}

// Store a message in MySQL database
function storeMessage(senderId, receiverId, message, callback) {
  const query =
    "INSERT INTO messages (sender_id, receiver_id, message, read_status) VALUES (?, ?, ?, ?)";
  db.query(query, [senderId, receiverId, message, 0], (err, result) => {
    if (err) throw err;
    callback(result);
  });
}

// WebSocket connection for real-time updates
io.on("connection", (socket) => {
  console.log("User connected");

  socket.on("subscribe", (userId) => {
    socket.join(userId);
    console.log(`User ${userId} subscribed to notifications`);
  });

  socket.on("disconnect", () => {
    console.log("User disconnected");
  });

  // Handle sending messages
  socket.on("sendMessage", (data) => {
    const { senderId, receiverId, message } = data;
    storeMessage(senderId, receiverId, message, (result) => {
      // Increment unread message count for receiver
      incrementUnreadMessages(receiverId, (newCount) => {
        // Notify the receiver about the new unread messages
        redisPublisher.publish(
          `notification:${receiverId}`,
          JSON.stringify({ unreadCount: newCount })
        );

        // Emit message to sender (optional, for sending real-time updates)
        io.to(senderId).emit("messageSent", {
          message: "Message sent successfully",
        });
      });
    });
  });

  // Mark messages as read
  socket.on("markAsRead", (userId) => {
    resetUnreadMessages(userId, (newCount) => {
      redisPublisher.publish(
        `notification:${userId}`,
        JSON.stringify({ unreadCount: newCount })
      );
    });
  });
});

// API endpoint to get unread message count for a user
app.get("/unread/:userId", (req, res) => {
  const userId = req.params.userId;
  getUnreadMessages(userId, (count) => {
    res.json({ unreadCount: count });
  });
});

// Start the server
server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

### Key Points:

1. **Redis Pub/Sub**:
   - Redis is used for real-time notifications. When a new message is sent to a user, it increments the unread count for that user.
   - A WebSocket message is published to the user’s Redis channel (`notification:userId`), which is received in real-time by the client.
2. **Message Storage**:
   - Each message is stored in the `messages` table of the MySQL database, with a `read_status` of `0` (unread).
3. **Message Sending**:

   - When a message is sent (`sendMessage` event), it is stored in the database, and the unread count for the receiver is incremented.

4. **Marking Messages as Read**:

   - When a user accesses the chat and marks messages as read, the unread count for that user is reset in Redis.

5. **Notification API**:
   - The `/unread/:userId` endpoint is used to check the unread message count for any user.

---

#### 2. **Frontend: React.js with `react-use-websocket`**

##### `App.js` (React.js)

```javascript
import React, { useEffect, useState } from "react";
import useWebSocket from "react-use-websocket";

const App = () => {
  const [userId, setUserId] = useState("1"); // This would be dynamically set in a real app
  const [unreadCount, setUnreadCount] = useState(0);
  const [messages, setMessages] = useState([]);

  // WebSocket connection to backend
  const { sendMessage, lastMessage, readyState } = useWebSocket(
    "ws://localhost:3000"
  );

  useEffect(() => {
    // Subscribe to notifications when the WebSocket connection is open
    if (readyState === 1) {
      sendMessage(JSON.stringify({ action: "subscribe", userId }));
    }
  }, [userId, readyState, sendMessage]);

  useEffect(() => {
    // Handle incoming unread count updates
    if (lastMessage !== null) {
      const { unreadCount } = JSON.parse(lastMessage.data);
      setUnreadCount(unreadCount);
    }
  }, [lastMessage]);

  // Fetch unread message count from backend
  const fetchUnreadCount = async () => {
    const res = await fetch(`http://localhost:3000/unread/${userId}`);
    const data = await res.json();
    setUnreadCount(data.unreadCount);
  };

  // Handle sending a message
  const sendMessageHandler = async () => {
    const message = "Hello, this is a test message!";
    sendMessage(JSON.stringify({ senderId: userId, receiverId: "2", message }));
  };

  return (
    <div>
      <h1>Chat Application</h1>

      <div>
        <button onClick={sendMessageHandler}>Send Message</button>
      </div>

      <div>
        <h2>Notifications</h2>
        <p>You have {unreadCount} unread messages</p>
      </div>

      <div>
        <h2>Messages</h2>
        <ul>
          {messages.map((msg, index) => (
            <li key={index}>{msg.message}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default App;
```

### Key Points:

1. **WebSocket Connection**: The React frontend connects to the backend via WebSocket to receive notifications in real-time. The app subscribes to the user’s specific notification channel.
2. **Unread Count**: The unread count is stored in the `unreadCount` state. This value is updated both via real-time notifications (via WebSocket) and by fetching from the backend API (`/unread/:userId`).

3. **Sending Messages**: The `sendMessageHandler` function is used to simulate sending a message. When a message is sent, the backend will increment the unread message count for the receiver.

4. **Real-Time Updates**: When a message is received, the unread count is updated on the frontend in real-time, and a notification badge can be displayed.

---

### Step-by-Step Execution:

1. **Backend**:

   - Install necessary dependencies:
     ```bash
     npm install express mysql redis socket.io http body-parser
     ```
   - Run the **Node.js backend**:
     ```bash
     node server.js
     ```

2. **Frontend**:
   - Install `react-use-websocket`:
     ```bash
     npm install react-use-websocket
     ```
   - Run the **React.js frontend**:
     ```bash
     npm start
     ```

---

### Conclusion:

This **notification system** ensures that users are notified of new unread messages, even if they are not currently in the chat.
It utilizes **Redis** for real-time communication, **MySQL/MSSQL** for storing messages, and **WebSockets** to push notifications in real-time.
The frontend React app listens for these updates and displays them with an unread message count, ensuring a smooth user experience in real-time messaging applications.
