# Microservices Architecture for Video Streaming Platform

## Overview
This documentation provides a detailed guide to implementing a **microservices architecture** for a video streaming platform. The architecture includes various independent services to handle authentication, video uploads, live streaming, real-time chats, and recommendation systems. These services communicate via REST APIs, and an **API Gateway** serves as the single entry point for all client requests.

---

## Architecture Diagram

1. **Auth Service**: Handles user authentication and authorization.
2. **Video Service**: Manages video uploads, HLS transcoding, and S3 storage.
3. **Live Streaming Service**: Manages WebRTC signaling for live streams.
4. **Chat Service**: Facilitates real-time chat using WebSockets.
5. **Recommendation Service**: Provides content recommendations based on user activity.
6. **API Gateway**: Acts as a reverse proxy to route requests to the appropriate service.

---

## Folder Structure

```plaintext
backend/
├── auth-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
├── video-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
├── live-stream-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── routes/
│   │   ├── signaling.js
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
├── chat-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── routes/
│   │   ├── websocket.js
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
├── recommendation-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
├── api-gateway/
│   ├── src/
│   │   ├── routes/
│   │   ├── app.js
│   │   ├── server.js
│   ├── package.json
└── docker-compose.yml
```

---

## Service Implementation Details

### 1. **Auth Service**

**Purpose**: Handles user registration, login, and JWT-based authentication.

#### Dependencies:
- `bcrypt`
- `jsonwebtoken`
- `express`
- `mongoose`

#### Example Routes:
- `POST /api/auth/register`: Registers a new user.
- `POST /api/auth/login`: Logs in a user and returns a JWT.

#### `auth-service/src/controllers/authController.js`
```javascript
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.registerUser = async (req, res) => {
  const { name, email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);

  const user = new User({ name, email, password: hashedPassword });
  await user.save();
  res.status(201).json({ message: 'User registered successfully' });
};

exports.loginUser = async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });

  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).json({ message: 'Invalid credentials' });
  }

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1d' });
  res.json({ token });
};
```

#### `auth-service/server.js`
```javascript
const express = require('express');
const mongoose = require('mongoose');
require('dotenv').config();

const app = express();
app.use(express.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('Auth Service DB Connected'));

app.use('/api/auth', require('./src/routes/auth'));

app.listen(5001, () => console.log('Auth Service running on port 5001'));
```

---

### 2. **Video Service**

**Purpose**: Handles video uploads and integration with AWS S3 for storage.

#### Dependencies:
- `aws-sdk`
- `multer`
- `express`

#### Example Routes:
- `POST /api/video/upload`: Uploads a video to S3.

#### `video-service/src/controllers/uploadController.js`
```javascript
const AWS = require('aws-sdk');
const multer = require('multer');
const storage = multer.memoryStorage();
const upload = multer({ storage });

const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
});

exports.uploadVideo = async (req, res) => {
  const params = {
    Bucket: process.env.AWS_BUCKET_NAME,
    Key: `${Date.now()}-${req.file.originalname}`,
    Body: req.file.buffer,
    ContentType: req.file.mimetype,
  };

  try {
    const data = await s3.upload(params).promise();
    res.json({ url: data.Location });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};
```

---

### 3. **Live Streaming Service**

**Purpose**: Manages WebRTC signaling for live streaming sessions.

#### Dependencies:
- `ws` (WebSocket library)
- `express`

#### Example Routes:
- `POST /api/live/start`: Initiates a live stream.
- `POST /api/live/join`: Joins an existing live stream.

---

### 4. **Chat Service**

**Purpose**: Implements real-time chat functionality using WebSockets.

#### Dependencies:
- `ws`

#### Example WebSocket Implementation:
- `ws://chat-service/messages`: Connects to the chat server for real-time communication.

---

### 5. **API Gateway**

**Purpose**: Serves as the single entry point for all client requests, routing them to appropriate services.

#### Dependencies:
- `http-proxy-middleware`

#### `api-gateway/server.js`
```javascript
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

app.use('/api/auth', createProxyMiddleware({ target: 'http://auth-service:5001', changeOrigin: true }));
app.use('/api/video', createProxyMiddleware({ target: 'http://video-service:5002', changeOrigin: true }));
app.use('/api/chat', createProxyMiddleware({ target: 'http://chat-service:5003', changeOrigin: true }));

app.listen(5000, () => console.log('API Gateway running on port 5000'));
```

---

## Deployment with Docker Compose

#### `docker-compose.yml`
```yaml
version: '3.8'
services:
  auth-service:
    build: ./auth-service
    ports:
      - "5001:5001"
    environment:
      - MONGO_URI=mongodb://mongo:27017/auth-db
      - JWT_SECRET=your-secret-key
  video-service:
    build: ./video-service
    ports:
      - "5002:5002"
    environment:
      - AWS_ACCESS_KEY_ID=your-access-key
      - AWS_SECRET_ACCESS_KEY=your-secret-key
      - AWS_BUCKET_NAME=your-bucket
  api-gateway:
    build: ./api-gateway
    ports:
      - "5000:5000"
  mongo:
    image: mongo
    ports:
      - "27017:27017"
```

---

## Advantages of Microservices Architecture
1. **Scalability**: Scale services independently.
2. **Fault Isolation**: Prevent system-wide crashes.
3. **Technology Flexibility**: Use different tech stacks for each service.
4. **Development Velocity**: Teams can work on different services concurrently.

---

## Conclusion
This documentation provides a foundation for implementing a robust, scalable video streaming platform using microservices. Each service is independently deployable and maintainable, making the architecture flexible for future enhancements.

