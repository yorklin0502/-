// anonymous-chat-app/server/server.js const express = require('express'); const http = require('http'); const { Server } = require('socket.io'); const cors = require('cors');

const app = express(); const server = http.createServer(app); const io = new Server(server, { cors: { origin: '*', methods: ['GET', 'POST'] } });

let waitingUsers = []; let activeChats = {};

io.on('connection', (socket) => { console.log(User connected: ${socket.id});

socket.on('join', (gender) => { socket.gender = gender; let match = waitingUsers.find(u => u.gender !== gender);

if (match) {
  waitingUsers = waitingUsers.filter(u => u.id !== match.id);
  const room = `${socket.id}#${match.id}`;
  activeChats[socket.id] = room;
  activeChats[match.id] = room;
  socket.join(room);
  io.to(match.id).emit('matched', room);
  socket.emit('matched', room);
  io.to(room).emit('message', 'You are now connected.');
} else {
  waitingUsers.push({ id: socket.id, gender });
}

});

socket.on('message', ({ room, message }) => { io.to(room).emit('message', message); });

socket.on('leave', () => { const room = activeChats[socket.id]; if (room) { io.to(room).emit('message', 'User left the chat.'); const [id1, id2] = room.split('#'); delete activeChats[id1]; delete activeChats[id2]; io.sockets.sockets.get(id1)?.leave(room); io.sockets.sockets.get(id2)?.leave(room); } });

socket.on('disconnect', () => { waitingUsers = waitingUsers.filter(u => u.id !== socket.id); const room = activeChats[socket.id]; if (room) { io.to(room).emit('message', 'User disconnected.'); const [id1, id2] = room.split('#'); delete activeChats[id1]; delete activeChats[id2]; } }); });

server.listen(3000, () => { console.log('Server running on http://localhost:3000'); });

// anonymous-chat-app/client/src/App.js import React, { useEffect, useState } from 'react'; import io from 'socket.io-client'; import firebase from 'firebase/app'; import 'firebase/auth'; import './App.css';

const socket = io('http://localhost:3000');

const firebaseConfig = { apiKey: 'YOUR_API_KEY', authDomain: 'YOUR_AUTH_DOMAIN', projectId: 'YOUR_PROJECT_ID', appId: 'YOUR_APP_ID' };

if (!firebase.apps.length) { firebase.initializeApp(firebaseConfig); }

function App() { const [user, setUser] = useState(null); const [gender, setGender] = useState('male'); const [room, setRoom] = useState(null); const [message, setMessage] = useState(''); const [messages, setMessages] = useState([]);

useEffect(() => { firebase.auth().onAuthStateChanged((u) => { if (u && u.email.endsWith('@cmu.edu.tw')) { setUser(u); } else { firebase.auth().signOut(); } });

socket.on('matched', (room) => {
  setRoom(room);
});

socket.on('message', (msg) => {
  setMessages((prev) => [...prev, msg]);
});

}, []);

const login = () => { const provider = new firebase.auth.GoogleAuthProvider(); firebase.auth().signInWithPopup(provider); };

const send = () => { if (message.trim() && room) { socket.emit('message', { room, message }); setMessage(''); } };

const joinChat = () => { socket.emit('join', gender); };

const leaveChat = () => { socket.emit('leave'); setRoom(null); setMessages([]); };

if (!user) return <button onClick={login}>Login with CMU Email</button>;

return ( <div className="App"> {!room ? ( <> <h2>Select Gender:</h2> <select value={gender} onChange={(e) => setGender(e.target.value)}> <option value="male">Male</option> <option value="female">Female</option> </select> <button onClick={joinChat}>Join Chat</button> </> ) : ( <> <div className="chat-box"> {messages.map((msg, idx) => ( <div key={idx}>{msg}</div> ))} </div> <input value={message} onChange={(e) => setMessage(e.target.value)} placeholder="Type message..." /> <button onClick={send}>Send</button> <button onClick={leaveChat}>Leave</button> </> )} </div> ); }

export default App;

// anonymous-chat-app/client/package.json { "name": "anonymous-chat-client", "version": "1.0.0", "private": true, "dependencies": { "firebase": "^9.0.0", "react": "^18.0.0", "react-dom": "^18.0.0", "react-scripts": "5.0.1", "socket.io-client": "^4.0.0" }, "scripts": { "start": "react-scripts start", "build": "react-scripts build" } }

// anonymous-chat-app/server/package.json { "name": "anonymous-chat-server", "version": "1.0.0", "main": "server.js", "scripts": { "start": "node server.js" }, "dependencies": { "cors": "^2.8.5", "express": "^4.17.1", "socket.io": "^4.0.0" } }

# -
