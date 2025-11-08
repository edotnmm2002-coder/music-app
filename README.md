# music-app
music streaming app witch all the artists need froming streaming to buying beats too make a banger on
# Music App - Full Starter (Backend + Frontend + CI)

> A complete starter codebase for a music streaming app with user accounts, music uploads, and paid subscriptions. License: GNU GPL v3.

---

## What this document contains

This document contains all the files you'll need to get started. Copy each file into your repository with the paths shown. Files included:

* `LICENSE`

* `README.md`

* `.gitignore`

* `backend/` (Node.js + Express API)

  * `package.json`
  * `index.js`
  * `routes/auth.js`
  * `routes/music.js`
  * `middleware/auth.js`
  * `stripe.js`
  * `uploads/` (local storage folder — gitignored)
  * `.env.example`

* `frontend/` (React + Vite)

  * `package.json`
  * `index.html`
  * `src/main.jsx`
  * `src/App.jsx`
  * `src/api.js`
  * `src/pages/Register.jsx`
  * `src/pages/Login.jsx`
  * `src/pages/Upload.jsx`
  * `src/pages/Player.jsx`
  * `src/styles.css`

* `.github/workflows/ci.yml` (GitHub Actions simple CI)

---

## LICENSE (GNU GPL v3)

<include the standard GPLv3 header — for brevity include a short notice here; replace with full text when publishing.>

Copyright (C) 2025

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

---

## README.md

````
# Music App

A music streaming app starter with user accounts, music uploads, and subscription support.

## Structure
- backend/ - Express API
- frontend/ - React app (Vite)

## Quickstart
1. Create two terminals.

### Backend
```bash
cd backend
cp .env.example .env
# fill in ENV values (JWT_SECRET, STRIPE_SECRET, etc.)
npm install
npm run dev
````

### Frontend

```bash
cd frontend
npm install
npm run dev
```

## Notes

* Uploads are stored in `backend/uploads` by default. For production, configure S3 (instructions in README).
* Stripe integration uses test keys in `.env`.

```

---

## .gitignore

```

node_modules/
uploads/
.env
dist/
build/
frontend/node_modules/
backend/node_modules/

````

---

## backend/package.json

```json
{
  "name": "music-app-backend",
  "version": "0.1.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "multer": "^1.4.5",
    "stripe": "^11.0.0",
    "sqlite3": "^5.1.6"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
````

---

## backend/.env.example

```
PORT=4000
JWT_SECRET=replace_with_a_strong_secret
DATABASE_FILE=./data.sqlite
STRIPE_SECRET_KEY=sk_test_replace
STRIPE_WEBHOOK_SECRET=whsec_replace
DOMAIN=http://localhost:5173
```

---

## backend/index.js

```js
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const authRoutes = require('./routes/auth');
const musicRoutes = require('./routes/music');
const stripe = require('./stripe');
const path = require('path');

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

// Serve uploaded files
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

app.use('/api/auth', authRoutes);
app.use('/api/music', musicRoutes);

// simple health
app.get('/api/health', (req, res) => res.json({ok: true}));

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Backend listening on ${port}`));
```

---

## backend/middleware/auth.js

```js
const jwt = require('jsonwebtoken');
const secret = process.env.JWT_SECRET || 'devsecret';

module.exports = function(req, res, next) {
  const header = req.headers.authorization;
  if (!header) return res.status(401).json({error: 'Missing authorization header'});
  const parts = header.split(' ');
  if (parts.length !== 2) return res.status(401).json({error: 'Invalid authorization header'});
  const token = parts[1];
  try {
    const payload = jwt.verify(token, secret);
    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).json({error: 'Invalid token'});
  }
};
```

---

## backend/routes/auth.js

```js
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');

const router = express.Router();
const secret = process.env.JWT_SECRET || 'devsecret';

async function getDb() {
  const db = await open({ filename: process.env.DATABASE_FILE || './data.sqlite', driver: sqlite3.Database });
  await db.exec(`CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, password TEXT, is_subscriber INTEGER DEFAULT 0)`);
  return db;
}

router.post('/register', async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password) return res.status(400).json({error: 'Missing email or password'});
  const db = await getDb();
  const hashed = await bcrypt.hash(password, 10);
  try {
    const result = await db.run('INSERT INTO users (email, password) VALUES (?, ?)', [email, hashed]);
    const user = { id: result.lastID, email };
    const token = jwt.sign(user, secret);
    res.json({ user, token });
  } catch (err) {
    res.status(400).json({ error: 'User exists or DB error' });
  }
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const db = await getDb();
  const row = await db.get('SELECT * FROM users WHERE email = ?', [email]);
  if (!row) return res.status(400).json({error: 'Invalid credentials'});
  const ok = await bcrypt.compare(password, row.password);
  if (!ok) return res.status(400).json({error: 'Invalid credentials'});
  const user = { id: row.id, email: row.email, is_subscriber: row.is_subscriber };
  const token = jwt.sign(user, secret);
  res.json({ user, token });
});

module.exports = router;
```

---

## backend/routes/music.js

```js
const express = require('express');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const auth = require('../middleware/auth');
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');

const router = express.Router();

const uploadDir = path.join(__dirname, '..', 'uploads');
if (!fs.existsSync(uploadDir)) fs.mkdirSync(uploadDir);
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, uploadDir)
  },
  filename: function (req, file, cb) {
    const safeName = Date.now() + '-' + file.originalname.replace(/[^a-zA-Z0-9.\-]/g, '_');
    cb(null, safeName)
  }
});
const upload = multer({ storage });

async function getDb() {
  const db = await open({ filename: process.env.DATABASE_FILE || './data.sqlite', driver: sqlite3.Database });
  await db.exec(`CREATE TABLE IF NOT EXISTS tracks (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT, filename TEXT, public INTEGER DEFAULT 1)`);
  return db;
}

// upload a track (authenticated)
router.post('/upload', auth, upload.single('track'), async (req, res) => {
  const db = await getDb();
  const userId = req.user.id;
  const title = req.body.title || req.file.originalname;
  const filename = req.file.filename;
  await db.run('INSERT INTO tracks (user_id, title, filename) VALUES (?, ?, ?)', [userId, title, filename]);
  res.json({ ok: true, filename, url: `/uploads/${filename}` });
});

// list tracks (public)
router.get('/', async (req, res) => {
  const db = await getDb();
  const rows = await db.all('SELECT t.id, t.title, t.filename, u.email as owner FROM tracks t JOIN users u ON u.id = t.user_id');
  res.json(rows.map(r => ({ ...r, url: `/uploads/${r.filename}` })));
});

module.exports = router;
```

---

## backend/stripe.js

```js
// simple server-side helpers for creating checkout sessions
const Stripe = require('stripe');
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || 'sk_test_replace');

async function createCheckoutSession(customerEmail, successUrl, cancelUrl) {
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'subscription',
    line_items: [{ price: 'price_replace_with_your_price_id', quantity: 1 }],
    customer_email: customerEmail,
    success_url: successUrl,
    cancel_url: cancelUrl,
  });
  return session;
}

module.exports = { createCheckoutSession };
```

> NOTE: Replace `price_replace_with_your_price_id` with the Stripe Price ID you create in your Stripe Dashboard. Use Stripe test keys for development.

---

## frontend/package.json

```json
{
  "name": "music-app-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.14.1"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

---

## frontend/index.html

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Music App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## frontend/src/main.jsx

```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<App />} />
      </Routes>
    </BrowserRouter>
  </React.StrictMode>
)
```

---

## frontend/src/App.jsx

```jsx
import React from 'react'
import { Link } from 'react-router-dom'
import Register from './pages/Register'
import Login from './pages/Login'
import Upload from './pages/Upload'
import Player from './pages/Player'

export default function App(){
  return (
    <div className="container">
      <h1>Music App</h1>
      <nav>
        <Link to="/register">Register</Link> | <Link to="/login">Login</Link> | <Link to="/upload">Upload</Link> | <Link to="/player">Player</Link>
      </nav>
      <main>
        <Register />
        <Login />
        <Upload />
        <Player />
      </main>
    </div>
  )
}
```

---

## frontend/src/api.js

```js
import axios from 'axios';
const API = axios.create({ baseURL: 'http://localhost:4000/api' });
export default API;
```

---

## frontend/src/pages/Register.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Register(){
  const [email,setEmail]=useState('');
  const [pw,setPw]=useState('');
  async function submit(e){
    e.preventDefault();
    const res = await API.post('/auth/register', { email, password: pw });
    alert('Registered — token: ' + (res.data.token ? 'received' : 'none'));
  }
  return (
    <section>
      <h2>Register</h2>
      <form onSubmit={submit}>
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="email" />
        <input value={pw} onChange={e=>setPw(e.target.value)} placeholder="password" type="password" />
        <button type="submit">Register</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Login.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Login(){
  const [email,setEmail]=useState('');
  const [pw,setPw]=useState('');
  async function submit(e){
    e.preventDefault();
    const res = await API.post('/auth/login', { email, password: pw });
    localStorage.setItem('token', res.data.token);
    alert('Logged in!');
  }
  return (
    <section>
      <h2>Login</h2>
      <form onSubmit={submit}>
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="email" />
        <input value={pw} onChange={e=>setPw(e.target.value)} placeholder="password" type="password" />
        <button type="submit">Login</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Upload.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Upload(){
  const [file,setFile]=useState(null);
  const [title,setTitle]=useState('');
  async function submit(e){
    e.preventDefault();
    const token = localStorage.getItem('token');
    if(!token){ alert('Please login'); return }
    const fd = new FormData();
    fd.append('track', file);
    fd.append('title', title);
    const res = await API.post('/music/upload', fd, { headers: { 'Authorization': 'Bearer ' + token, 'Content-Type': 'multipart/form-data' } });
    alert('Uploaded: ' + JSON.stringify(res.data));
  }
  return (
    <section>
      <h2>Upload</h2>
      <form onSubmit={submit}>
        <input type="text" value={title} onChange={e=>setTitle(e.target.value)} placeholder="title" />
        <input type="file" accept="audio/*" onChange={e=>setFile(e.target.files[0])} />
        <button type="submit">Upload</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Player.jsx

```jsx
import React, {useEffect, useState} from 'react';
import API from '../api';

export default function Player(){
  const [tracks,setTracks]=useState([]);
  useEffect(()=>{ API.get('/music').then(r=>setTracks(r.data)) },[]);
  return (
    <section>
      <h2>Player</h2>
      <ul>
        {tracks.map(t=> (
          <li key={t.id}>
            <div>{t.title} — {t.owner}</div>
            <audio controls src={ 'http://localhost:4000' + t.url } />
          </li>
        ))}
      </ul>
    </section>
  )
}
```

---

## frontend/src/styles.css

```css
body{ font-family: Arial, sans-serif; padding: 20px }
.container{ max-width: 900px; margin: 0 auto }
nav a{ margin-right: 10px }
section{ border: 1px solid #eee; padding: 12px; margin-top: 12px }
```

---

## .github/workflows/ci.yml

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install backend deps
        run: |
          cd backend
          npm ci
      - name: Run backend lint/test placeholder
        run: echo "No tests yet"

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install frontend deps
        run: |
          cd frontend
          npm ci
      - name: Build frontend
        run: |
          cd frontend
          npm run build
# Music App - Full Starter (Backend + Frontend + CI)

> A complete starter codebase for a music streaming app with user accounts, music uploads, and paid subscriptions. License: GNU GPL v3.

---

## What this document contains

This document contains all the files you'll need to get started. Copy each file into your repository with the paths shown. Files included:

* `LICENSE`

* `README.md`

* `.gitignore`

* `backend/` (Node.js + Express API)

  * `package.json`
  * `index.js`
  * `routes/auth.js`
  * `routes/music.js`
  * `middleware/auth.js`
  * `stripe.js`
  * `uploads/` (local storage folder — gitignored)
  * `.env.example`

* `frontend/` (React + Vite)

  * `package.json`
  * `index.html`
  * `src/main.jsx`
  * `src/App.jsx`
  * `src/api.js`
  * `src/pages/Register.jsx`
  * `src/pages/Login.jsx`
  * `src/pages/Upload.jsx`
  * `src/pages/Player.jsx`
  * `src/styles.css`

* `.github/workflows/ci.yml` (GitHub Actions simple CI)

---

## LICENSE (GNU GPL v3)

<include the standard GPLv3 header — for brevity include a short notice here; replace with full text when publishing.>

Copyright (C) 2025

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

---

## README.md

````
# Music App

A music streaming app starter with user accounts, music uploads, and subscription support.

## Structure
- backend/ - Express API
- frontend/ - React app (Vite)

## Quickstart
1. Create two terminals.

### Backend
```bash
cd backend
cp .env.example .env
# fill in ENV values (JWT_SECRET, STRIPE_SECRET, etc.)
npm install
npm run dev
````

### Frontend

```bash
cd frontend
npm install
npm run dev
```

## Notes

* Uploads are stored in `backend/uploads` by default. For production, configure S3 (instructions in README).
* Stripe integration uses test keys in `.env`.

```

---

## .gitignore

```

node_modules/
uploads/
.env
dist/
build/
frontend/node_modules/
backend/node_modules/

````

---

## backend/package.json

```json
{
  "name": "music-app-backend",
  "version": "0.1.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "multer": "^1.4.5",
    "stripe": "^11.0.0",
    "sqlite3": "^5.1.6"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
````

---

## backend/.env.example

```
PORT=4000
JWT_SECRET=replace_with_a_strong_secret
DATABASE_FILE=./data.sqlite
STRIPE_SECRET_KEY=sk_test_replace
STRIPE_WEBHOOK_SECRET=whsec_replace
DOMAIN=http://localhost:5173
```

---

## backend/index.js

```js
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const authRoutes = require('./routes/auth');
const musicRoutes = require('./routes/music');
const stripe = require('./stripe');
const path = require('path');

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

// Serve uploaded files
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

app.use('/api/auth', authRoutes);
app.use('/api/music', musicRoutes);

// simple health
app.get('/api/health', (req, res) => res.json({ok: true}));

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Backend listening on ${port}`));
```

---

## backend/middleware/auth.js

```js
const jwt = require('jsonwebtoken');
const secret = process.env.JWT_SECRET || 'devsecret';

module.exports = function(req, res, next) {
  const header = req.headers.authorization;
  if (!header) return res.status(401).json({error: 'Missing authorization header'});
  const parts = header.split(' ');
  if (parts.length !== 2) return res.status(401).json({error: 'Invalid authorization header'});
  const token = parts[1];
  try {
    const payload = jwt.verify(token, secret);
    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).json({error: 'Invalid token'});
  }
};
```

---

## backend/routes/auth.js

```js
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');

const router = express.Router();
const secret = process.env.JWT_SECRET || 'devsecret';

async function getDb() {
  const db = await open({ filename: process.env.DATABASE_FILE || './data.sqlite', driver: sqlite3.Database });
  await db.exec(`CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, password TEXT, is_subscriber INTEGER DEFAULT 0)`);
  return db;
}

router.post('/register', async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password) return res.status(400).json({error: 'Missing email or password'});
  const db = await getDb();
  const hashed = await bcrypt.hash(password, 10);
  try {
    const result = await db.run('INSERT INTO users (email, password) VALUES (?, ?)', [email, hashed]);
    const user = { id: result.lastID, email };
    const token = jwt.sign(user, secret);
    res.json({ user, token });
  } catch (err) {
    res.status(400).json({ error: 'User exists or DB error' });
  }
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const db = await getDb();
  const row = await db.get('SELECT * FROM users WHERE email = ?', [email]);
  if (!row) return res.status(400).json({error: 'Invalid credentials'});
  const ok = await bcrypt.compare(password, row.password);
  if (!ok) return res.status(400).json({error: 'Invalid credentials'});
  const user = { id: row.id, email: row.email, is_subscriber: row.is_subscriber };
  const token = jwt.sign(user, secret);
  res.json({ user, token });
});

module.exports = router;
```

---

## backend/routes/music.js

```js
const express = require('express');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const auth = require('../middleware/auth');
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');

const router = express.Router();

const uploadDir = path.join(__dirname, '..', 'uploads');
if (!fs.existsSync(uploadDir)) fs.mkdirSync(uploadDir);
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, uploadDir)
  },
  filename: function (req, file, cb) {
    const safeName = Date.now() + '-' + file.originalname.replace(/[^a-zA-Z0-9.\-]/g, '_');
    cb(null, safeName)
  }
});
const upload = multer({ storage });

async function getDb() {
  const db = await open({ filename: process.env.DATABASE_FILE || './data.sqlite', driver: sqlite3.Database });
  await db.exec(`CREATE TABLE IF NOT EXISTS tracks (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT, filename TEXT, public INTEGER DEFAULT 1)`);
  return db;
}

// upload a track (authenticated)
router.post('/upload', auth, upload.single('track'), async (req, res) => {
  const db = await getDb();
  const userId = req.user.id;
  const title = req.body.title || req.file.originalname;
  const filename = req.file.filename;
  await db.run('INSERT INTO tracks (user_id, title, filename) VALUES (?, ?, ?)', [userId, title, filename]);
  res.json({ ok: true, filename, url: `/uploads/${filename}` });
});

// list tracks (public)
router.get('/', async (req, res) => {
  const db = await getDb();
  const rows = await db.all('SELECT t.id, t.title, t.filename, u.email as owner FROM tracks t JOIN users u ON u.id = t.user_id');
  res.json(rows.map(r => ({ ...r, url: `/uploads/${r.filename}` })));
});

module.exports = router;
```

---

## backend/stripe.js

```js
// simple server-side helpers for creating checkout sessions
const Stripe = require('stripe');
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || 'sk_test_replace');

async function createCheckoutSession(customerEmail, successUrl, cancelUrl) {
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'subscription',
    line_items: [{ price: 'price_replace_with_your_price_id', quantity: 1 }],
    customer_email: customerEmail,
    success_url: successUrl,
    cancel_url: cancelUrl,
  });
  return session;
}

module.exports = { createCheckoutSession };
```

> NOTE: Replace `price_replace_with_your_price_id` with the Stripe Price ID you create in your Stripe Dashboard. Use Stripe test keys for development.

---

## frontend/package.json

```json
{
  "name": "music-app-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.14.1"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

---

## frontend/index.html

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Music App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## frontend/src/main.jsx

```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<App />} />
      </Routes>
    </BrowserRouter>
  </React.StrictMode>
)
```

---

## frontend/src/App.jsx

```jsx
import React from 'react'
import { Link } from 'react-router-dom'
import Register from './pages/Register'
import Login from './pages/Login'
import Upload from './pages/Upload'
import Player from './pages/Player'

export default function App(){
  return (
    <div className="container">
      <h1>Music App</h1>
      <nav>
        <Link to="/register">Register</Link> | <Link to="/login">Login</Link> | <Link to="/upload">Upload</Link> | <Link to="/player">Player</Link>
      </nav>
      <main>
        <Register />
        <Login />
        <Upload />
        <Player />
      </main>
    </div>
  )
}
```

---

## frontend/src/api.js

```js
import axios from 'axios';
const API = axios.create({ baseURL: 'http://localhost:4000/api' });
export default API;
```

---

## frontend/src/pages/Register.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Register(){
  const [email,setEmail]=useState('');
  const [pw,setPw]=useState('');
  async function submit(e){
    e.preventDefault();
    const res = await API.post('/auth/register', { email, password: pw });
    alert('Registered — token: ' + (res.data.token ? 'received' : 'none'));
  }
  return (
    <section>
      <h2>Register</h2>
      <form onSubmit={submit}>
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="email" />
        <input value={pw} onChange={e=>setPw(e.target.value)} placeholder="password" type="password" />
        <button type="submit">Register</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Login.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Login(){
  const [email,setEmail]=useState('');
  const [pw,setPw]=useState('');
  async function submit(e){
    e.preventDefault();
    const res = await API.post('/auth/login', { email, password: pw });
    localStorage.setItem('token', res.data.token);
    alert('Logged in!');
  }
  return (
    <section>
      <h2>Login</h2>
      <form onSubmit={submit}>
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="email" />
        <input value={pw} onChange={e=>setPw(e.target.value)} placeholder="password" type="password" />
        <button type="submit">Login</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Upload.jsx

```jsx
import React, {useState} from 'react';
import API from '../api';

export default function Upload(){
  const [file,setFile]=useState(null);
  const [title,setTitle]=useState('');
  async function submit(e){
    e.preventDefault();
    const token = localStorage.getItem('token');
    if(!token){ alert('Please login'); return }
    const fd = new FormData();
    fd.append('track', file);
    fd.append('title', title);
    const res = await API.post('/music/upload', fd, { headers: { 'Authorization': 'Bearer ' + token, 'Content-Type': 'multipart/form-data' } });
    alert('Uploaded: ' + JSON.stringify(res.data));
  }
  return (
    <section>
      <h2>Upload</h2>
      <form onSubmit={submit}>
        <input type="text" value={title} onChange={e=>setTitle(e.target.value)} placeholder="title" />
        <input type="file" accept="audio/*" onChange={e=>setFile(e.target.files[0])} />
        <button type="submit">Upload</button>
      </form>
    </section>
  )
}
```

---

## frontend/src/pages/Player.jsx

```jsx
import React, {useEffect, useState} from 'react';
import API from '../api';

export default function Player(){
  const [tracks,setTracks]=useState([]);
  useEffect(()=>{ API.get('/music').then(r=>setTracks(r.data)) },[]);
  return (
    <section>
      <h2>Player</h2>
      <ul>
        {tracks.map(t=> (
          <li key={t.id}>
            <div>{t.title} — {t.owner}</div>
            <audio controls src={ 'http://localhost:4000' + t.url } />
          </li>
        ))}
      </ul>
    </section>
  )
}
```

---

## frontend/src/styles.css

```css
body{ font-family: Arial, sans-serif; padding: 20px }
.container{ max-width: 900px; margin: 0 auto }
nav a{ margin-right: 10px }
section{ border: 1px solid #eee; padding: 12px; margin-top: 12px }
```

---

## .github/workflows/ci.yml

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install backend deps
        run: |
          cd backend
          npm ci
      - name: Run backend lint/test placeholder
        run: echo "No tests yet"

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install frontend deps
        run: |
          cd frontend
          npm ci
      - name: Build frontend
        run: |
          cd frontend
          npm run build

1. **Database:** This starter uses SQLite for simplicity. For production use Postgres or MySQL.
2. **Storage:** Switch uploads to S3 (or similar) for scalability.
3. **Payments:** Create Stripe Price IDs and replace placeholders. Securely store secrets.
4. **Security:** Use HTTPS in production, validate uploaded files and sizes, add rate limiting.
5. **Testing:** Add unit and integration tests.

---

If you want, I can now:

* Generate each file as separate commits and provide exact `git` commands to run locally.
* Swap the backend to Flask/Django or change the frontend stack.
* Add S3 upload example and Stripe webhook handling.

Tell me which of those you'd like me to do next.












