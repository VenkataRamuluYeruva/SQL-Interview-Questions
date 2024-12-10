### WebRTC for Audio and Video Calling: Full Stack Implementation

WebRTC (Web Real-Time Communication) is a protocol that allows peer-to-peer communication directly between browsers. It facilitates real-time communication without requiring plugins. WebRTC can be used for video, audio calling, and file sharing. For a production-level system that supports multiple users, we'll use WebSockets for signaling (to exchange messages like offer, answer, ICE candidates) and WebRTC for the actual peer-to-peer communication.

Here's a breakdown of the process and how you can implement WebRTC in both the backend (Node.js) and frontend (React.js) using `react-use-websocket`.

### Overview of Steps

1. **Signaling Server**: You need a WebSocket server to handle signaling between clients (exchanging offers, answers, and ICE candidates).
2. **WebRTC Connection**: Each client (browser) will establish a WebRTC connection with others. This will involve creating an RTC peer connection, setting up media (audio/video), and exchanging ICE candidates.
3. **Multiple User Support**: We'll implement a multi-user chatroom-like feature where any participant can join and leave the call.

### 1. **Backend - Node.js with WebSocket**

We'll use `ws`, a WebSocket library for Node.js, to manage connections between clients.

#### Install dependencies:

```bash
npm install ws express
```

#### WebSocket Signaling Server Code (`server.js`):

```javascript
const WebSocket = require("ws");
const express = require("express");
const app = express();
const http = require("http");
const server = http.createServer(app);

// Create a WebSocket server
const wss = new WebSocket.Server({ server });

const clients = new Map(); // Store WebSocket connections by user ID

// Handle incoming WebSocket connections
wss.on("connection", (ws) => {
  let userId;
  ws.on("message", (message) => {
    const data = JSON.parse(message);

    switch (data.type) {
      case "join":
        userId = data.userId;
        clients.set(userId, ws);
        console.log(`${userId} joined`);
        break;

      case "offer":
      case "answer":
      case "candidate":
        const recipient = data.targetId;
        const targetClient = clients.get(recipient);
        if (targetClient) {
          targetClient.send(JSON.stringify(data));
        }
        break;

      case "leave":
        clients.delete(userId);
        console.log(`${userId} left`);
        break;

      default:
        break;
    }
  });

  ws.on("close", () => {
    if (userId) {
      clients.delete(userId);
      console.log(`${userId} disconnected`);
    }
  });
});

const port = 5000;
server.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
```

### Explanation:

- **WebSocket connection**: Each client connects to this server using WebSockets.
- **Signaling**: The server listens for "offer", "answer", and "candidate" messages and forwards them to the appropriate target.
- **User management**: We use a `Map` to store users and their respective WebSocket connections.

### 2. **Frontend - React.js with WebRTC**

#### Install dependencies:

```bash
npm install react-use-websocket
npm install react-dom react-scripts
```

#### Basic WebRTC Video Calling App (`App.js`):

```jsx
import React, { useState, useEffect, useRef } from "react";
import useWebSocket from "react-use-websocket";

const App = () => {
  const [isJoined, setIsJoined] = useState(false);
  const [userId, setUserId] = useState("");
  const [peerConnections, setPeerConnections] = useState({});
  const [localStream, setLocalStream] = useState(null);

  const { sendMessage, lastMessage } = useWebSocket("ws://localhost:5000", {
    onOpen: () => console.log("WebSocket Connected"),
    onMessage: handleSignalingMessage,
  });

  const localVideoRef = useRef(null);
  const remoteVideoRef = useRef(null);

  useEffect(() => {
    navigator.mediaDevices
      .getUserMedia({ audio: true, video: true })
      .then((stream) => {
        setLocalStream(stream);
        localVideoRef.current.srcObject = stream;
      })
      .catch((err) => console.error("Failed to get media: ", err));
  }, []);

  // Handle signaling messages
  const handleSignalingMessage = (event) => {
    const message = JSON.parse(event.data);
    switch (message.type) {
      case "offer":
        handleOffer(message);
        break;
      case "answer":
        handleAnswer(message);
        break;
      case "candidate":
        handleCandidate(message);
        break;
      default:
        break;
    }
  };

  const handleOffer = (message) => {
    const peerConnection = new RTCPeerConnection();
    peerConnection.setRemoteDescription(
      new RTCSessionDescription(message.offer)
    );

    localStream.getTracks().forEach((track) => {
      peerConnection.addTrack(track, localStream);
    });

    peerConnection.createAnswer().then((answer) => {
      peerConnection.setLocalDescription(answer);
      sendMessage(
        JSON.stringify({
          type: "answer",
          targetId: message.userId,
          answer,
        })
      );
    });

    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        sendMessage(
          JSON.stringify({
            type: "candidate",
            targetId: message.userId,
            candidate: event.candidate,
          })
        );
      }
    };

    peerConnection.ontrack = (event) => {
      remoteVideoRef.current.srcObject = event.streams[0];
    };

    setPeerConnections((prevState) => ({
      ...prevState,
      [message.userId]: peerConnection,
    }));
  };

  const handleAnswer = (message) => {
    const peerConnection = peerConnections[message.userId];
    peerConnection.setRemoteDescription(
      new RTCSessionDescription(message.answer)
    );
  };

  const handleCandidate = (message) => {
    const peerConnection = peerConnections[message.userId];
    peerConnection.addIceCandidate(new RTCIceCandidate(message.candidate));
  };

  // Join the call
  const joinCall = () => {
    setIsJoined(true);
    setUserId(`user-${Date.now()}`);
    sendMessage(JSON.stringify({ type: "join", userId }));
  };

  // Start call (Offer to other users)
  const startCall = (targetId) => {
    const peerConnection = new RTCPeerConnection();
    localStream.getTracks().forEach((track) => {
      peerConnection.addTrack(track, localStream);
    });

    peerConnection.createOffer().then((offer) => {
      peerConnection.setLocalDescription(offer);
      sendMessage(
        JSON.stringify({
          type: "offer",
          targetId,
          offer,
        })
      );
    });

    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        sendMessage(
          JSON.stringify({
            type: "candidate",
            targetId,
            candidate: event.candidate,
          })
        );
      }
    };

    peerConnection.ontrack = (event) => {
      remoteVideoRef.current.srcObject = event.streams[0];
    };

    setPeerConnections((prevState) => ({
      ...prevState,
      [targetId]: peerConnection,
    }));
  };

  return (
    <div>
      {!isJoined && <button onClick={joinCall}>Join Call</button>}

      {isJoined && (
        <>
          <button onClick={() => startCall("user-1234")}>Call User 1234</button>
          <video ref={localVideoRef} autoPlay muted />
          <video ref={remoteVideoRef} autoPlay />
        </>
      )}
    </div>
  );
};

export default App;
```

### Explanation:

- **WebRTC Setup**: This code establishes a WebRTC connection using `RTCPeerConnection`. It handles `offer`, `answer`, and `candidate` messages for signaling.
- **Media Streams**: We capture the local video/audio stream using `getUserMedia()` and display it in the `localVideoRef`. The remote stream is shown in `remoteVideoRef`.
- **WebSocket Signaling**: The `react-use-websocket` library handles WebSocket connections and sends/receives signaling messages (like offers, answers, ICE candidates).
- **Peer Connection**: When starting a call, a peer connection is created and tracks are added to it. Offers and answers are exchanged via WebSocket.

### 3. **Run the Application**

1. **Start the WebSocket Server (Node.js)**:

```bash
node server.js
```

2. **Run the React Application**:

```bash
npm start
```

This will launch the frontend on `http://localhost:3000` and the WebSocket server on `ws://localhost:5000`.

### 4. **Handling Multiple Users**

- When a user joins, they can start calling another user (as in `startCall('user-1234')`). Multiple users can join and create peer connections with each other.
- To support multiple users more efficiently, you'd need to scale the WebSocket server and handle each participant's peer connections properly.

### 5. **Conclusion**

- **WebRTC** enables peer-to-peer communication, and **WebSockets** are used for signaling.
- This implementation allows multiple users to connect and start a video/audio call with each other.
- In a production system, you'd need to handle network issues, user disconnections, and other edge cases. You'd also need to ensure the WebSocket server can handle a large number of connections if your app grows in scale.

### Detailed Explanation of the Code

The code is a React app that facilitates WebRTC-based video/audio calling with WebSocket signaling. Below, I’ll break down the code step-by-step and explain the role of each part, along with how you can modify or extend it for additional features.

### 1. **State Management (`useState`)**

```jsx
const [isJoined, setIsJoined] = useState(false);
const [userId, setUserId] = useState("");
const [peerConnections, setPeerConnections] = useState({});
const [localStream, setLocalStream] = useState(null);
```

- **isJoined**: Tracks whether the user has joined the call.
- **userId**: Stores the user’s unique identifier.
- **peerConnections**: A dictionary of peer connections keyed by user IDs (this allows multiple connections).
- **localStream**: Stores the local media stream (video and audio captured from the user's camera and microphone).

### 2. **WebSocket Connection (`useWebSocket`)**

```jsx
const { sendMessage, lastMessage } = useWebSocket("ws://localhost:5000", {
  onOpen: () => console.log("WebSocket Connected"),
  onMessage: handleSignalingMessage,
});
```

- **useWebSocket**: This hook establishes a WebSocket connection with the signaling server (`ws://localhost:5000`).
  - `sendMessage`: Function to send messages (like offers, answers, and ICE candidates).
  - `onMessage`: A callback that triggers when a message is received via WebSocket. It calls `handleSignalingMessage` to process the incoming message.

### 3. **Video Elements (`useRef`)**

```jsx
const localVideoRef = useRef(null);
const remoteVideoRef = useRef(null);
```

- `localVideoRef`: A reference to the `<video>` element for showing the user's local video stream.
- `remoteVideoRef`: A reference to the `<video>` element for showing the remote video stream from the other user.

### 4. **Fetching Media Stream (`getUserMedia`)**

```jsx
useEffect(() => {
  navigator.mediaDevices
    .getUserMedia({ audio: true, video: true })
    .then((stream) => {
      setLocalStream(stream);
      localVideoRef.current.srcObject = stream;
    })
    .catch((err) => console.error("Failed to get media: ", err));
}, []);
```

- This `useEffect` hook requests the user's media (video and audio) when the component mounts.
- `getUserMedia`: A browser API that accesses the user’s camera and microphone.
- The local stream is set to the state (`setLocalStream`) and displayed in the local video element (`localVideoRef`).

### 5. **Handling Signaling Messages**

```jsx
const handleSignalingMessage = (event) => {
  const message = JSON.parse(event.data);
  switch (message.type) {
    case "offer":
      handleOffer(message);
      break;
    case "answer":
      handleAnswer(message);
      break;
    case "candidate":
      handleCandidate(message);
      break;
    default:
      break;
  }
};
```

- This function processes incoming signaling messages via WebSocket.
- The signaling types can be:
  - **offer**: An offer to start a call (from the caller).
  - **answer**: A response to an offer (from the receiver).
  - **candidate**: ICE candidates used for establishing the peer connection.

### 6. **Handling the Offer (Incoming Call)**

```jsx
const handleOffer = (message) => {
  const peerConnection = new RTCPeerConnection();
  peerConnection.setRemoteDescription(new RTCSessionDescription(message.offer));

  localStream.getTracks().forEach((track) => {
    peerConnection.addTrack(track, localStream);
  });

  peerConnection.createAnswer().then((answer) => {
    peerConnection.setLocalDescription(answer);
    sendMessage(
      JSON.stringify({
        type: "answer",
        targetId: message.userId,
        answer,
      })
    );
  });

  peerConnection.onicecandidate = (event) => {
    if (event.candidate) {
      sendMessage(
        JSON.stringify({
          type: "candidate",
          targetId: message.userId,
          candidate: event.candidate,
        })
      );
    }
  };

  peerConnection.ontrack = (event) => {
    remoteVideoRef.current.srcObject = event.streams[0];
  };

  setPeerConnections((prevState) => ({
    ...prevState,
    [message.userId]: peerConnection,
  }));
};
```

- **RTCPeerConnection**: This establishes a connection to the remote peer.
- **setRemoteDescription**: Set the received offer (SDP).
- **localStream.getTracks().forEach**: Add the local media tracks (audio and video) to the peer connection.
- **createAnswer**: Generate an answer in response to the offer.
- **onicecandidate**: If new ICE candidates are found, send them to the other peer.
- **ontrack**: When a remote track (video/audio) is received, display it in the remote video element.

### 7. **Handling the Answer (Call Accepted)**

```jsx
const handleAnswer = (message) => {
  const peerConnection = peerConnections[message.userId];
  peerConnection.setRemoteDescription(
    new RTCSessionDescription(message.answer)
  );
};
```

- Once the offer is accepted, the peer connection’s remote description is set to the received answer.

### 8. **Handling ICE Candidate (Network Information)**

```jsx
const handleCandidate = (message) => {
  const peerConnection = peerConnections[message.userId];
  peerConnection.addIceCandidate(new RTCIceCandidate(message.candidate));
};
```

- The ICE candidate is received and added to the peer connection to maintain connectivity (handling network traversal).

### 9. **Join the Call**

```jsx
const joinCall = () => {
  setIsJoined(true);
  setUserId(`user-${Date.now()}`);
  sendMessage(JSON.stringify({ type: "join", userId }));
};
```

- When the user clicks "Join Call", they are assigned a unique `userId` (timestamp).
- This `userId` is sent to the signaling server so that the user can join the call.

### 10. **Start the Call (Making an Offer)**

```jsx
const startCall = (targetId) => {
  const peerConnection = new RTCPeerConnection();
  localStream.getTracks().forEach((track) => {
    peerConnection.addTrack(track, localStream);
  });

  peerConnection.createOffer().then((offer) => {
    peerConnection.setLocalDescription(offer);
    sendMessage(
      JSON.stringify({
        type: "offer",
        targetId,
        offer,
      })
    );
  });

  peerConnection.onicecandidate = (event) => {
    if (event.candidate) {
      sendMessage(
        JSON.stringify({
          type: "candidate",
          targetId,
          candidate: event.candidate,
        })
      );
    }
  };

  peerConnection.ontrack = (event) => {
    remoteVideoRef.current.srcObject = event.streams[0];
  };

  setPeerConnections((prevState) => ({
    ...prevState,
    [targetId]: peerConnection,
  }));
};
```

- The user clicks "Call User 1234", which triggers the `startCall` function.
- An offer is created and sent to the target user.

### 11. **UI Rendering**

```jsx
return (
  <div>
    {!isJoined && <button onClick={joinCall}>Join Call</button>}
    {isJoined && (
      <>
        <button onClick={() => startCall("user-1234")}>Call User 1234</button>
        <video ref={localVideoRef} autoPlay muted />
        <video ref={remoteVideoRef} autoPlay />
      </>
    )}
  </div>
);
```

- **Join Call**: The user can click the "Join Call" button to join the call.
- **Start Call**: After joining, the user can start calling another user (for example, `user-1234`).
- **Video Elements**: Displays the local and remote video streams.

---

### How to Modify the Code for Extra Features:

1. **Add Chat Feature**:

   - You can extend the WebSocket signaling to support text chat.
   - Modify the WebSocket communication to send chat messages, and create an additional text input field for sending and receiving messages.

2. **Multiple Participants**:

   - Currently, the app only handles one-to-one calls. For multiple users, you’d need to:
     - Maintain a list of active peer connections for each participant.
     - Handle multiple incoming offers and distribute answers/candidates to all peers.
     - Use a layout that dynamically adds video elements for each participant (grid layout).

3. **Handle User Disconnects**:

   - Track when users leave or disconnect, and clean up their peer connections.
   - You can add functionality like a "Leave Call" button and send a disconnect message via WebSocket.

4. **Video and Audio Controls**:

   - Add features to mute/unmute the microphone and video or switch between video sources (e.g., front/back camera).
   - Implement buttons for muting the audio and turning off the video (`localStream.getTracks()` can be used to control each media track).

5. **Error Handling**:

   - Add more robust error handling, especially for cases like network issues, ICE connection failures, etc.

6. **Turn Servers for NAT Traversal**:
   - In production, you should set up **STUN** and **TURN** servers for better NAT traversal (connecting users behind routers/firewalls). These can be specified in the `RTCPeerConnection` configuration.

By modifying and expanding on these elements, you can add various features to create a fully-functional WebRTC-based video/audio call application.
