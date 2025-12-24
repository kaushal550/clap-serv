# Step-by-Step GitHub Deployment Guide for Clap-Serv

## 🎯 Stack Overview
- **Frontend**: React (HTML/JS)
- **Backend**: Node.js + Express
- **Database**: PostgreSQL
- **File Storage**: Google Cloud Storage (instead of AWS S3)
- **Maps**: Google Maps API
- **Authentication**: Google OAuth (optional) + JWT
- **Hosting**: Google Cloud Platform or any VPS

---

## 📦 Step 1: Set Up GitHub Repository

### 1.1 Create GitHub Account (if you don't have one)
```bash
# Go to https://github.com and sign up
```

### 1.2 Install Git on Your Computer
```bash
# Windows: Download from https://git-scm.com/download/win
# Mac: 
brew install git

# Linux (Ubuntu/Debian):
sudo apt update
sudo apt install git

# Verify installation
git --version
```

### 1.3 Configure Git
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### 1.4 Create New Repository on GitHub
1. Go to https://github.com
2. Click the "+" icon in top right
3. Select "New repository"
4. Name it: `clap-serv`
5. Description: "Community Services Marketplace Platform"
6. Choose: Public or Private
7. ✅ Check "Add a README file"
8. Click "Create repository"

### 1.5 Clone Repository to Your Computer
```bash
# Copy your repository URL from GitHub (green "Code" button)
git clone https://github.com/YOUR-USERNAME/clap-serv.git
cd clap-serv
```

---

## 📁 Step 2: Project Structure Setup

### 2.1 Create Project Folders
```bash
# From inside clap-serv folder
mkdir -p frontend/public
mkdir -p frontend/src
mkdir -p backend/src/config
mkdir -p backend/src/models
mkdir -p backend/src/routes
mkdir -p backend/src/controllers
mkdir -p backend/src/middleware
mkdir -p backend/src/utils
```

### 2.2 Your Project Structure Should Look Like:
```
clap-serv/
├── frontend/
│   ├── public/
│   │   ├── index.html
│   │   └── logo.png
│   ├── src/
│   └── package.json
├── backend/
│   ├── src/
│   │   ├── config/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── middleware/
│   │   └── utils/
│   ├── .env.example
│   └── package.json
├── .gitignore
└── README.md
```

### 2.3 Copy Your Files
```bash
# Copy the index.html I created to frontend/public/
# Copy logo.png to frontend/public/

# We'll create backend files step by step below
```

---

## 🔧 Step 3: Backend Setup with Node.js + PostgreSQL

### 3.1 Initialize Backend
```bash
cd backend
npm init -y
```

### 3.2 Install Dependencies
```bash
# Core dependencies
npm install express cors helmet compression dotenv
npm install pg sequelize
npm install jsonwebtoken bcryptjs
npm install multer
npm install @google-cloud/storage
npm install socket.io

# Development dependencies
npm install --save-dev nodemon
```

### 3.3 Create package.json Scripts
Edit `backend/package.json` and add:
```json
{
  "name": "clap-serv-backend",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "compression": "^1.7.4",
    "dotenv": "^16.3.1",
    "pg": "^8.11.3",
    "sequelize": "^6.35.1",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "multer": "^1.4.5-lts.1",
    "@google-cloud/storage": "^7.7.0",
    "socket.io": "^4.6.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

### 3.4 Create .env.example File
Create `backend/.env.example`:
```env
# Server Configuration
NODE_ENV=development
PORT=5000
FRONTEND_URL=http://localhost:3000

# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=clapserv
DB_USER=your_db_user
DB_PASSWORD=your_db_password

# JWT Secret
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production
JWT_EXPIRE=7d

# Google Cloud Configuration
GOOGLE_APPLICATION_CREDENTIALS=./google-cloud-key.json
GCS_BUCKET_NAME=clap-serv-uploads

# Google Maps API
GOOGLE_MAPS_API_KEY=your-google-maps-api-key

# Email Configuration (using Gmail)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
```

### 3.5 Create .env File (Don't Commit This!)
```bash
cp .env.example .env
# Edit .env with your actual credentials
```

---

## 🗄️ Step 4: PostgreSQL Database Setup

### 4.1 Install PostgreSQL

**Windows:**
1. Download from https://www.postgresql.org/download/windows/
2. Run installer
3. Set password for postgres user
4. Note the port (default: 5432)

**Mac:**
```bash
brew install postgresql@14
brew services start postgresql@14
```

**Linux (Ubuntu):**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### 4.2 Create Database
```bash
# Access PostgreSQL
sudo -u postgres psql

# Inside psql terminal:
CREATE DATABASE clapserv;
CREATE USER clapserv_user WITH ENCRYPTED PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE clapserv TO clapserv_user;
\q
```

### 4.3 Create Database Configuration File
Create `backend/src/config/database.js`:
```javascript
const { Sequelize } = require('sequelize');
require('dotenv').config();

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    dialect: 'postgres',
    logging: false,
    pool: {
      max: 5,
      min: 0,
      acquire: 30000,
      idle: 10000
    }
  }
);

// Test connection
sequelize
  .authenticate()
  .then(() => {
    console.log('✅ Database connection established successfully.');
  })
  .catch((err) => {
    console.error('❌ Unable to connect to database:', err);
  });

module.exports = sequelize;
```

### 4.4 Create Database Models

**User Model** - `backend/src/models/User.js`:
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');
const bcrypt = require('bcryptjs');

const User = sequelize.define('User', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  email: {
    type: DataTypes.STRING,
    allowNull: false,
    unique: true,
    validate: {
      isEmail: true
    }
  },
  password: {
    type: DataTypes.STRING,
    allowNull: false
  },
  activeRole: {
    type: DataTypes.STRING,
    defaultValue: 'buyer',
    validate: {
      isIn: [['buyer', 'provider', 'admin']]
    }
  },
  phone: {
    type: DataTypes.STRING
  },
  locationLat: {
    type: DataTypes.DECIMAL(10, 8)
  },
  locationLng: {
    type: DataTypes.DECIMAL(11, 8)
  },
  bio: {
    type: DataTypes.TEXT
  },
  rating: {
    type: DataTypes.DECIMAL(3, 2),
    defaultValue: 0
  },
  reviews: {
    type: DataTypes.INTEGER,
    defaultValue: 0
  },
  completedJobs: {
    type: DataTypes.INTEGER,
    defaultValue: 0
  },
  responseTime: {
    type: DataTypes.STRING
  },
  address: {
    type: DataTypes.STRING
  },
  homepage: {
    type: DataTypes.STRING
  },
  gigs: {
    type: DataTypes.JSONB,
    defaultValue: []
  },
  youtubeVideos: {
    type: DataTypes.JSONB,
    defaultValue: []
  },
  services: {
    type: DataTypes.JSONB,
    defaultValue: []
  }
}, {
  tableName: 'users',
  timestamps: true
});

// Hash password before saving
User.beforeCreate(async (user) => {
  if (user.password) {
    const salt = await bcrypt.genSalt(10);
    user.password = await bcrypt.hash(user.password, salt);
  }
});

// Method to compare password
User.prototype.comparePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

module.exports = User;
```

**Service Category Model** - `backend/src/models/ServiceCategory.js`:
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const ServiceCategory = sequelize.define('ServiceCategory', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  icon: {
    type: DataTypes.STRING
  },
  description: {
    type: DataTypes.TEXT
  },
  distanceRange: {
    type: DataTypes.STRING,
    allowNull: false
  }
}, {
  tableName: 'service_categories',
  timestamps: true
});

module.exports = ServiceCategory;
```

**Service Request Model** - `backend/src/models/ServiceRequest.js`:
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const ServiceRequest = sequelize.define('ServiceRequest', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  buyerId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'users',
      key: 'id'
    }
  },
  categoryId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'service_categories',
      key: 'id'
    }
  },
  title: {
    type: DataTypes.STRING,
    allowNull: false
  },
  description: {
    type: DataTypes.TEXT,
    allowNull: false
  },
  images: {
    type: DataTypes.JSONB,
    defaultValue: []
  },
  videos: {
    type: DataTypes.JSONB,
    defaultValue: []
  },
  locationLat: {
    type: DataTypes.DECIMAL(10, 8)
  },
  locationLng: {
    type: DataTypes.DECIMAL(11, 8)
  },
  status: {
    type: DataTypes.STRING,
    defaultValue: 'open'
  }
}, {
  tableName: 'service_requests',
  timestamps: true
});

module.exports = ServiceRequest;
```

**Proposal Model** - `backend/src/models/Proposal.js`:
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Proposal = sequelize.define('Proposal', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  requestId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'service_requests',
      key: 'id'
    }
  },
  providerId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'users',
      key: 'id'
    }
  },
  bidAmount: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false
  },
  message: {
    type: DataTypes.TEXT
  },
  status: {
    type: DataTypes.STRING,
    defaultValue: 'pending'
  }
}, {
  tableName: 'proposals',
  timestamps: true
});

module.exports = Proposal;
```

**Message Model** - `backend/src/models/Message.js`:
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Message = sequelize.define('Message', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  proposalId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'proposals',
      key: 'id'
    }
  },
  senderId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'users',
      key: 'id'
    }
  },
  message: {
    type: DataTypes.TEXT,
    allowNull: false
  }
}, {
  tableName: 'messages',
  timestamps: true
});

module.exports = Message;
```

**Index Models File** - `backend/src/models/index.js`:
```javascript
const User = require('./User');
const ServiceCategory = require('./ServiceCategory');
const ServiceRequest = require('./ServiceRequest');
const Proposal = require('./Proposal');
const Message = require('./Message');

// Define associations
ServiceRequest.belongsTo(User, { foreignKey: 'buyerId', as: 'buyer' });
ServiceRequest.belongsTo(ServiceCategory, { foreignKey: 'categoryId' });

Proposal.belongsTo(ServiceRequest, { foreignKey: 'requestId' });
Proposal.belongsTo(User, { foreignKey: 'providerId', as: 'provider' });

Message.belongsTo(Proposal, { foreignKey: 'proposalId' });
Message.belongsTo(User, { foreignKey: 'senderId', as: 'sender' });

module.exports = {
  User,
  ServiceCategory,
  ServiceRequest,
  Proposal,
  Message
};
```

---

## 🚀 Step 5: Create Backend Server

### 5.1 Main Server File - `backend/src/server.js`:
```javascript
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
require('dotenv').config();

const sequelize = require('./config/database');
const models = require('./models');

const app = express();
const server = http.createServer(app);
const io = socketIO(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    methods: ['GET', 'POST']
  }
});

// Middleware
app.use(helmet());
app.use(cors());
app.use(compression());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/services', require('./routes/services'));
app.use('/api/requests', require('./routes/requests'));
app.use('/api/proposals', require('./routes/proposals'));

// Socket.IO for real-time chat
io.on('connection', (socket) => {
  console.log('User connected:', socket.id);
  
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    console.log(`User ${socket.id} joined room ${roomId}`);
  });
  
  socket.on('send-message', (data) => {
    io.to(data.roomId).emit('receive-message', data);
  });
  
  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'OK', message: 'Clap-Serv API is running' });
});

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ 
    error: 'Something went wrong!',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined
  });
});

// Sync database and start server
const PORT = process.env.PORT || 5000;

sequelize.sync({ alter: true }).then(() => {
  console.log('✅ Database synced');
  server.listen(PORT, () => {
    console.log(`🚀 Server running on port ${PORT}`);
  });
}).catch((err) => {
  console.error('❌ Database sync error:', err);
});
```

### 5.2 Create Authentication Routes - `backend/src/routes/auth.js`:
```javascript
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const { User } = require('../models');

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ where: { email } });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    // Create user
    const user = await User.create({
      name,
      email,
      password,
      activeRole: 'buyer'
    });
    
    // Generate token
    const token = jwt.sign(
      { id: user.id, email: user.email },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      message: 'User created successfully',
      token,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        activeRole: user.activeRole
      }
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Server error' });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Find user
    const user = await User.findOne({ where: { email } });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Check password
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate token
    const token = jwt.sign(
      { id: user.id, email: user.email },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      message: 'Login successful',
      token,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        activeRole: user.activeRole,
        services: user.services,
        rating: user.rating,
        reviews: user.reviews
      }
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Server error' });
  }
});

module.exports = router;
```

### 5.3 Create Authentication Middleware - `backend/src/middleware/auth.js`:
```javascript
const jwt = require('jsonwebtoken');
const { User } = require('../models');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findByPk(decoded.id);
    
    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }
    
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

module.exports = auth;
```

---

## ☁️ Step 6: Google Cloud Setup

### 6.1 Create Google Cloud Project
1. Go to https://console.cloud.google.com
2. Create new project: "Clap-Serv"
3. Enable billing (required for APIs)

### 6.2 Enable Required APIs
```
1. Google Maps JavaScript API
2. Google Maps Geocoding API
3. Google Maps Distance Matrix API
4. Google Cloud Storage API
```

### 6.3 Create API Keys
1. Go to "APIs & Services" > "Credentials"
2. Click "Create Credentials" > "API Key"
3. Copy the API key
4. Restrict the key (recommended):
   - Application restrictions: HTTP referrers (for frontend)
   - API restrictions: Select only needed APIs

### 6.4 Set Up Google Cloud Storage
1. Go to "Cloud Storage" > "Buckets"
2. Click "Create Bucket"
3. Name: `clap-serv-uploads`
4. Choose location
5. Create

### 6.5 Create Service Account
1. Go to "IAM & Admin" > "Service Accounts"
2. Create service account: "clap-serv-storage"
3. Grant role: "Storage Object Admin"
4. Create key (JSON)
5. Download and save as `google-cloud-key.json` in backend folder

### 6.6 Configure Google Cloud Storage - `backend/src/config/storage.js`:
```javascript
const { Storage } = require('@google-cloud/storage');
const multer = require('multer');
const path = require('path');

// Initialize Google Cloud Storage
const storage = new Storage({
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
  projectId: 'your-project-id'
});

const bucket = storage.bucket(process.env.GCS_BUCKET_NAME);

// Multer configuration for file upload
const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 5 * 1024 * 1024 // 5MB limit
  }
});

// Upload file to Google Cloud Storage
const uploadToGCS = async (file) => {
  const blob = bucket.file(`uploads/${Date.now()}-${file.originalname}`);
  const blobStream = blob.createWriteStream({
    resumable: false,
    metadata: {
      contentType: file.mimetype
    }
  });

  return new Promise((resolve, reject) => {
    blobStream.on('error', (err) => reject(err));
    blobStream.on('finish', () => {
      const publicUrl = `https://storage.googleapis.com/${bucket.name}/${blob.name}`;
      resolve(publicUrl);
    });
    blobStream.end(file.buffer);
  });
};

module.exports = { upload, uploadToGCS };
```

---

## 📤 Step 7: Push to GitHub

### 7.1 Create .gitignore File
Create `.gitignore` in root folder:
```
# Dependencies
node_modules/
frontend/node_modules/
backend/node_modules/

# Environment variables
.env
backend/.env
frontend/.env

# Google Cloud credentials
google-cloud-key.json
*.json

# Build files
frontend/build/
frontend/dist/

# Logs
*.log
npm-debug.log*

# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

### 7.2 Add Files to Git
```bash
# From root clap-serv folder
git add .
git commit -m "Initial commit: Clap-Serv platform setup"
```

### 7.3 Push to GitHub
```bash
git branch -M main
git push -u origin main
```

### 7.4 Verify on GitHub
Go to your repository on GitHub and verify all files are uploaded.

---

## 🌐 Step 8: Deploy to Production

### Option A: Deploy to Google Cloud Platform (Recommended)

```bash
# Install Google Cloud SDK
# Follow: https://cloud.google.com/sdk/docs/install

# Login
gcloud auth login

# Set project
gcloud config set project YOUR_PROJECT_ID

# Deploy backend
cd backend
gcloud app deploy

# Deploy frontend
cd ../frontend
npm run build
gcloud app deploy
```

### Option B: Deploy to Heroku (Free Tier)

```bash
# Install Heroku CLI
# https://devcenter.heroku.com/articles/heroku-cli

# Login
heroku login

# Create app
heroku create clap-serv-api

# Add PostgreSQL
heroku addons:create heroku-postgresql:hobby-dev

# Set environment variables
heroku config:set JWT_SECRET=your-secret
heroku config:set GOOGLE_MAPS_API_KEY=your-key

# Deploy
git push heroku main
```

---

## ✅ Step 9: Testing

### 9.1 Test Backend
```bash
cd backend
npm run dev

# Test in browser or Postman:
# http://localhost:5000/health
```

### 9.2 Test Frontend
```bash
cd frontend
# Open index.html in browser
# Or use a local server:
npx http-server public
```

---

## 📝 Next Steps

1. ✅ Add more routes for requests, proposals, messages
2. ✅ Implement Google Maps integration in frontend
3. ✅ Add file upload functionality
4. ✅ Set up automated tests
5. ✅ Configure CI/CD pipeline
6. ✅ Add monitoring and logging
7. ✅ Set up domain and SSL

---

## 🆘 Troubleshooting

**Problem: Cannot connect to database**
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Check credentials in .env file
```

**Problem: Git push rejected**
```bash
# Pull latest changes first
git pull origin main
git push origin main
```

**Problem: Google Cloud Storage upload fails**
```bash
# Verify service account JSON file path
# Check bucket permissions
```

---

## 📞 Support

For questions:
- GitHub Issues: https://github.com/YOUR-USERNAME/clap-serv/issues
- Email: your-email@example.com

---

**You're all set! 🎉**

This guide will take you from zero to a fully deployed Clap-Serv platform using GitHub, Google APIs, and PostgreSQL!
