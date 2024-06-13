## **System Design**

![System Architecture Diagram](./images/Blank%20diagram.jpeg)

## Architecture Diagram

The architecture diagram consists of the following components:

- **Client Application**: User interface for interacting with the quiz
- **WebSocket Server**: Handles real-time communication between clients and server
- **Backend Server**: Manages business logic, quiz management, and score updates
- **Database**: Stores user information, quiz data, and scores
- **Cache**: Stores real-time leaderboard updates and reduces database load

## Component Description

- **Client Application**: Web or mobile app for users to join quiz sessions, submit answers, and view the leaderboard
- **WebSocket Server**: Enables real-time bidirectional communication between client and server
- **Backend Server**: Manages user sessions, processes quiz logic, calculates scores, and updates the leaderboard
- **Database**: Stores persistent data such as user details, quiz questions, and scores
- **Cache**: Speeds up access to frequently updated data such as the leaderboard

## Data Flow

1.  User joins a quiz by entering a unique quiz ID via the Client Application
2.  Client Application establishes a WebSocket connection to the WebSocket Server
3.  WebSocket Server communicates with the Backend Server to validate the quiz ID and join the user to the session
4.  User submits answers via WebSocket
5.  WebSocket Server forwards answers to the Backend Server for processing
6.  Backend Server calculates the score and updates the user's score in the Database
7.  Updated leaderboard is stored in the Cache and sent back to the WebSocket Server
8.  WebSocket Server broadcasts the updated leaderboard to all connected clients

## Technologies and Tools

- **Client Application**: React.js (web) or React Native (mobile)
- **WebSocket Server**: Node.js with the `ws` library
- **Backend Server**: Express.js (Node.js framework)
- **Database**: PostgreSQL
- **Cache**: Redis

## Implementation

The implementation focuses on the WebSocket Server component using Node.js and the `ws` library. The code example demonstrates how to handle user connections, answer submissions, and leaderboard updates in real-time.

### Step 1:

Initialize a new Node.js project and install the following packages:

`npm install ws express redis`

### Step 2:

Create a file `websocket-server.js`:

    const WebSocket = require('ws');
    const express = require('express');
    const http = require('http');
    const Redis = require('ioredis');

    const app = express();
    const server = http.createServer(app);
    const wss = new WebSocket.Server({ server });

    const redis = new Redis();

    wss.on('connection', (ws) => {
      ws.on('message', async (message) => {
        const { type, data } = JSON.parse(message);

        if (type === 'join') {
          const { quizId, userId } = data;
          // Handle user joining quiz
          ws.quizId = quizId;
          ws.userId = userId;
          // Add user to quiz in Redis
          await redis.sadd(`quiz:${quizId}:users`, userId);
          // Broadcast updated leaderboard
          broadcastLeaderboard(quizId);
        } else if (type === 'answer') {
          const { quizId, userId, answer } = data;
          // Handle user answer submission
          const score = calculateScore(answer); // Placeholder function
          await redis.hincrby(`quiz:${quizId}:scores`, userId, score);
          // Broadcast updated leaderboard
          broadcastLeaderboard(quizId);
        }
      });
    });

    const broadcastLeaderboard = async (quizId) => {
      const users = await redis.smembers(`quiz:${quizId}:users`);
      const scores = await redis.hgetall(`quiz:${quizId}:scores`);

      const leaderboard = users.map(user => ({
        userId: user,
        score: scores[user] || 0
      })).sort((a, b) => b.score - a.score);

      wss.clients.forEach(client => {
        if (client.readyState === WebSocket.OPEN && client.quizId === quizId) {
          client.send(JSON.stringify({ type: 'leaderboard', data: leaderboard }));
        }
      });
    };

    const calculateScore = (answer) => {
      // Implement your scoring logic here
      return answer === 'correct' ? 10 : 0;
    };

    server.listen(8080, () => {
      console.log('WebSocket server is listening on port 8080');
    });

### Step 3:

Run the WebSocket Server

`node websocket-server.js`

## Final Thoughts:

This implementation centers around the WebSocket Server, which enables real-time communication and handles multiple users joining quiz sessions, submitting answers, and dynamically updating the leaderboard. Future developments will involve creating the Client Application and Backend API, and integrating the entire system with robust error handling, security measures, and scalable architecture.
