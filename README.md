# Arkeolojik-veri-taban-
arkeolojik veri tabanı
arkeoloji-veritabani/
│
├── backend/
│   ├── models/
│   │   └── User.js
│   ├── auth.js
│   ├── index.js
│   ├── package.json
│   └── .env.example
│
├── frontend/
│   ├── pages/
│   │   ├── login.js
│   │   └── register.js
│   ├── public/
│   ├── next.config.js
│   ├── package.json
│   └── README.md
│
└── README.md
{
  "name": "arkeoloji-backend",
  "version": "1.0.0",
  "description": "Backend for arkeoloji veri tabanı",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "mongoose": "^7.3.1"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
MONGODB_URI=mongodb://localhost:27017/arkeoloji
JWT_SECRET=supersecretkey123
PORT=5000
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const { router: authRouter } = require('./auth');

const app = express();
app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error(err));

app.use('/api/auth', authRouter);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const router = express.Router();

const User = require('./models/User');

const JWT_SECRET = process.env.JWT_SECRET || 'secretkey123';

// Register
router.post('/register', async (req, res) => {
  try {
    const { username, password, role } = req.body;

    if (!username || !password) return res.status(400).json({ message: 'Eksik bilgi' });

    const existingUser = await User.findOne({ username });
    if (existingUser) return res.status(400).json({ message: 'Kullanıcı zaten var' });

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = new User({ username, password: hashedPassword, role: role || 'user' });
    await user.save();

    res.status(201).json({ message: 'Kayıt başarılı' });
  } catch (error) {
    res.status(500).json({ message: 'Sunucu hatası', error });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;

    const user = await User.findOne({ username });
    if (!user) return res.status(400).json({ message: 'Kullanıcı bulunamadı' });

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) return res.status(400).json({ message: 'Parola hatalı' });

    const token = jwt.sign({ id: user._id, role: user.role }, JWT_SECRET, { expiresIn: '1d' });

    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Sunucu hatası', error });
  }
});

// Auth middleware
const authMiddleware = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader) return res.status(401).json({ message: 'Yetkisiz' });

  const token = authHeader.split(' ')[1];
  if (!token) return res.status(401).json({ message: 'Yetkisiz' });

  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    return res.status(401).json({ message: 'Token geçersiz' });
  }
};

module.exports = { router, authMiddleware };
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' }
});

module.exports = mongoose.model('User', UserSchema);
{
  "name": "arkeoloji-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "^13.5.1",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://localhost:5000/api/:path*'
      }
    ];
  }
};
import { useState } from 'react';
import { useRouter } from 'next/router';

export default function Register() {
  const [form, setForm] = useState({ username: '', password: '' });
  const [message, setMessage] = useState('');
  const router = useRouter();

  const handleChange = (e) => {
    setForm({...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    const res = await fetch('/api/auth/register', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(form)
    });
    const data = await res.json();
    setMessage(data.message);
    if (res.ok) {
      router.push('/login');
    }
  };

  return (
    <div style={{ maxWidth: '400px', margin: 'auto', paddingTop: '50px' }}>
      <h1>Kayıt Ol</h1>
      <form onSubmit={handleSubmit}>
        <input name="username" placeholder="Kullanıcı Adı" value={form.username} onChange={handleChange} required style={{width:'100%', marginBottom:'10px', padding:'8px'}} />
        <input name="password" type="password" placeholder="Parola" value={form.password} onChange={handleChange} required style={{width:'100%', marginBottom:'10px', padding:'8px'}} />
        <button type="submit" style={{width:'100%', padding:'10px'}}>Kayıt Ol</button>
      </form>
      <p>{message}</p>
    </div>
  );
}
import { useState } from 'react';
import { useRouter } from 'next/router';

export default function Login() {
  const [form, setForm] = useState({ username: '', password: '' });
  const [message, setMessage] = useState('');
  const router = useRouter();

  const handleChange = (e) => {
    setForm({...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(form)
    });
    const data = await res.json();
    if (res.ok) {
      localStorage.setItem('token', data.token);
      router.push('/');
    } else {
      setMessage(data.message);
    }
  };

  return (
    <div style={{ maxWidth: '400px', margin: 'auto', paddingTop: '50px' }}>
      <h1>Giriş Yap</h1>
      <form onSubmit={handleSubmit}>
        <input name="username" placeholder="Kullanıcı Adı" value={form.username} onChange={handleChange} required style={{width:'100%', marginBottom:'10px', padding:'8px'}} />
        <input name="password" type="password" placeholder="Parola" value={form.password} onChange={handleChange} required style={{width:'100%', marginBottom:'10px', padding:'8px'}} />
        <button type="submit" style={{width:'100%', padding:'10px'}}>Giriş Yap</button>
      </form>
      <p>{message}</p>
    </div>
  );
}
# Arkeolojik Veri Tabanı Projesi

## Proje Yapısı

- `backend/` - Node.js + Express backend uygulaması  
- `frontend/` - Next.js React frontend uygulaması  

## Kurulum

### Backend

1. `backend` klasörüne girin:

2. Gerekli paketleri yükleyin:

3. `.env` dosyasını oluşturun ve MongoDB bağlantınızı ayarlayın (`.env.example` dosyasına bakabilirsiniz).

4. MongoDB çalıştırın (lokal ya da uzaktaki bağlantı).

5. Sunucuyu başlatın:

### Frontend

1. `frontend` klasörüne girin:

2. Gerekli paketleri yükleyin:

3. Next.js uygulamasını başlatın:

4. Tarayıcıda `http://localhost:3000/register` veya `/login` sayfalarına giderek test edin.

---

## Notlar

- Frontend, backend API isteklerini `http://localhost:5000/api` adresine proxy yapmaktadır.
- Backend üzerinde JWT ile kullanıcı kimlik doğrulaması yapılmaktadır.
- Bu aşamada sadece kullanıcı kayıt ve giriş işlemleri mevcuttur.
- İlerleyen adımlarda eser arama, filtreleme, admin paneli ve diğer modüller eklenecektir.
