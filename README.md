# coding-project
for learning 
#include <stdio.h>

int main() {
    printf("Hello, Linux!\n");
    return 0;
}#include <stdio.h>

int main() {
    printf("Hello, Linux!\n");
    return 0;
}#include <stdio.h>

int main() {
    printf("Hello, Linux!\n");
    return 0;
}gcc hello.c -o hello
./hello
chmod +x hello.sh
./hello.sh
{
  "name": "dating-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "sequelize": "^6.32.1",
    "sqlite3": "^5.1.6",
    "socket.io": "^4.8.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
const express = require('express');
const http = require('http');
const cors = require('cors');
const { Sequelize, DataTypes, Op } = require('sequelize');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { Server } = require('socket.io');

const SECRET = process.env.JWT_SECRET || 'replace-with-secure-secret';
const PORT = process.env.PORT || 4000;

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

app.use(cors());
app.use(express.json());

// SQLite + Sequelize
const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: './database.sqlite',
  logging: false,
});

// Models
const User = sequelize.define('User', {
  name: { type: DataTypes.STRING, allowNull: false },
  email: { type: DataTypes.STRING, unique: true, allowNull: false },
  passwordHash: { type: DataTypes.STRING, allowNull: false },
  bio: { type: DataTypes.TEXT },
  gender: { type: DataTypes.STRING },
  dob: { type: DataTypes.DATEONLY },
  photos: { type: DataTypes.TEXT }, // store JSON stringified array of urls
}, {});

const Like = sequelize.define('Like', {
  fromId: { type: DataTypes.INTEGER, allowNull: false },
  toId: { type: DataTypes.INTEGER, allowNull: false },
}, {});

const Match = sequelize.define('Match', {
  userA: { type: DataTypes.INTEGER, allowNull: false },
  userB: { type: DataTypes.INTEGER, allowNull: false },
}, {});

const Message = sequelize.define('Message', {
  matchId: { type: DataTypes.INTEGER, allowNull: false },
  senderId: { type: DataTypes.INTEGER, allowNull: false },
  text: { type: DataTypes.TEXT, allowNull: false },
}, { timestamps: true });

// Utility
const generateToken = (user) => jwt.sign({ id: user.id, email: user.email }, SECRET, { expiresIn: '7d' });

const authMiddleware = async (req, res, next) => {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ error: 'Missing auth header' });
  const token = auth.split(' ')[1];
  try {
    const payload = jwt.verify(token, SECRET);
    const user = await User.findByPk(payload.id);
    if (!user) return res.status(401).json({ error: 'User not found' });
    req.user = user;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Routes
app.post('/api/register', async (req, res) => {
  const { name, email, password } = req.body;
  if (!name || !email || !password) return res.status(400).json({ error: 'Missing fields' });
  try {
    const passwordHash = await bcrypt.hash(password, 10);
    const user = await User.create({ name, email, passwordHash, photos: JSON.stringify([]) });
    const token = generateToken(user);
    res.json({ token, user: { id: user.id, name: user.name, email: user.email } });
  } catch (err) {
    res.status(400).json({ error: 'Email already in use or invalid' });
  }
});

app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ where: { email }});
  if (!user) return res.status(400).json({ error: 'Invalid credentials' });
  const ok = await bcrypt.compare(password, user.passwordHash);
  if (!ok) return res.status(400).json({ error: 'Invalid credentials' });
  const token = generateToken(user);
  res.json({ token, user: { id: user.id, name: user.name, email: user.email } });
});

// Get own profile
app.get('/api/me', authMiddleware, async (req, res) => {
  const user = req.user.toJSON();
  user.photos = JSON.parse(user.photos || '[]');
  delete user.passwordHash;
  res.json(user);
});

// Update profile
app.put('/api/me', authMiddleware, async (req, res) => {
  const { name, bio, gender, dob, photos } = req.body;
  const user = req.user;
  await user.update({ name, bio, gender, dob, photos: JSON.stringify(photos || []) });
  const u = user.toJSON();
  u.photos = JSON.parse(u.photos || '[]');
  delete u.passwordHash;
  res.json(u);
});

// Simple discovery endpoint: returns users you haven't liked or matched
app.get('/api/discover', authMiddleware, async (req, res) => {
  const myId = req.user.id;

  // exclude self, those already liked by me or I liked them, and matched ones
  const likedTo = await Like.findAll({ where: { fromId: myId }});
  const likedIds = likedTo.map(l => l.toId);

  const matches = await Match.findAll({ where: { [Op.or]: [{ userA: myId }, { userB: myId }] }});
  const matchedIds = matches.flatMap(m => (m.userA === myId ? [m.userB] : [m.userA]));

  const exclude = [myId, ...likedIds, ...matchedIds];

  const candidates = await User.findAll({
    where: { id: { [Op.notIn]: exclude } },
    order: sequelize.random(),
    limit: 20
  });

  const simplified = candidates.map(u => {
    const p = u.toJSON();
    p.photos = JSON.parse(p.photos || '[]');
    delete p.passwordHash;
    return p;
  });

  res.json(simplified);
});

// Like endpoint
app.post('/api/like', authMiddleware, async (req, res) => {
  const fromId = req.user.id;
  const { toId } = req.body;
  if (!toId) return res.status(400).json({ error: 'Missing toId' });
  if (fromId === toId) return res.status(400).json({ error: 'Cannot like yourself' });

  // record like
  const existing = await Like.findOne({ where: { fromId, toId }});
  if (!existing) await Like.create({ fromId, toId });

  // check reciprocal
  const reciprocal = await Like.findOne({ where: { fromId: toId, toId: fromId }});
  if (reciprocal) {
    // create match (unique)
    const alreadyMatched = await Match.findOne({ where: {
      [Op.or]: [
        { userA: fromId, userB: toId },
        { userA: toId, userB: fromId }
      ]
    }});
    if (!alreadyMatched) {
      const match = await Match.create({ userA: fromId, userB: toId });
      // notify via socket
      io.to(String(fromId)).emit('match', { matchId: match.id, otherId: toId });
      io.to(String(toId)).emit('match', { matchId: match.id, otherId: fromId });
      return res.json({ matched: true, matchId: match.id });
    }
  }

  res.json({ matched: false });
});

// List matches
app.get('/api/matches', authMiddleware, async (req, res) => {
  const myId = req.user.id;
  const matches = await Match.findAll({ where: { [Op.or]: [{ userA: myId }, { userB: myId }] }});
  const results = [];
  for (const m of matches) {
    const otherId = m.userA === myId ? m.userB : m.userA;
    const user = await User.findByPk(otherId);
    const u = user.toJSON(); u.photos = JSON.parse(u.photos || '[]'); delete u.passwordHash;
    results.push({ matchId: m.id, user: u, createdAt: m.createdAt });
  }
  res.json(results);
});

// Messages for a match
app.get('/api/messages/:matchId', authMiddleware, async (req, res) => {
  const myId = req.user.id;
  const matchId = Number(req.params.matchId);
  const m = await Match.findByPk(matchId);
  if (!m) return res.status(404).json({ error: 'Match not found' });
  if (!(m.userA === myId || m.userB === myId)) return res.status(403).json({ error: 'Not part of match' });
  const messages = await Message.findAll({ where: { matchId }, order: [['createdAt','ASC']] });
  res.json(messages);
});

// send message (also via socket)
app.post('/api/messages/:matchId', authMiddleware, async (req, res) => {
  const myId = req.user.id;
  const matchId = Number(req.params.matchId);
  const { text } = req.body;
  if (!text) return res.status(400).json({ error: 'Empty message' });
  const m = await Match.findByPk(matchId);
  if (!m) return res.status(404).json({ error: 'Match not found' });
  if (!(m.userA === myId || m.userB === myId)) return res.status(403).json({ error: 'Not part of match' });

  const message = await Message.create({ matchId, senderId: myId, text });
  // send via socket to both participants (rooms are user IDs)
  const otherId = m.userA === myId ? m.userB : m.userA;
  io.to(String(otherId)).emit('message', { matchId, message });
  io.to(String(myId)).emit('message', { matchId, message });

  res.json(message);
});

// Init DB and start
(async () => {
  await sequelize.sync({ alter: true });
  server.listen(PORT, () => console.log('Server listening on', PORT));
})();

// Socket.io: use userId as room
io.on('connection', (socket) => {
  // client should send auth token on connect query or after connection
  const { token } = socket.handshake.query;
  if (!token) return;
  try {
    const payload = jwt.verify(token, SECRET);
    const userId = payload.id;
    socket.join(String(userId));
    socket.on('disconnect', () => {
      socket.leave(String(userId));
    });
  } catch (err) {
    socket.disconnect(true);
  }
        })
