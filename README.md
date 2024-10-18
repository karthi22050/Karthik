npm init -y
npm install express mongoose bcryptjs jsonwebtoken nodemailer dotenv body-parser
MONGO_URI=mongodb://localhost:27017/jobportal
JWT_SECRET=your_jwt_secret
EMAIL_USER=your_email@example.com
EMAIL_PASS=your_email_password
const express = require('express');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const authRoutes = require('./routes/auth');
const jobRoutes = require('./routes/jobs');
require('dotenv').config();

const app = express();
app.use(bodyParser.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

app.use('/auth', authRoutes); // Routes for registration, login
app.use('/jobs', jobRoutes);   // Routes for job posting, emails

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const CompanySchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  mobile: { type: String, required: true },
  password: { type: String, required: true },
  verified: { type: Boolean, default: false },
});

CompanySchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

module.exports = mongoose.model('Company', CompanySchema);

const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const Company = require('../models/Company');
const nodemailer = require('nodemailer');
const router = express.Router();

// Company Registration
router.post('/register', async (req, res) => {
  const { name, email, mobile, password } = req.body;
  
  try {
    let company = await Company.findOne({ email });
    if (company) return res.status(400).json({ msg: 'Company already exists' });
    
    company = new Company({ name, email, mobile, password });
    await company.save();
    
    const token = jwt.sign({ id: company._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    // Send verification email (optional for now)
    // sendVerificationEmail(company, token);
    
    res.json({ token });
  } catch (err) {
    res.status(500).send('Server error');
  }
});

// Company Login
router.post('/login', async (req, res) => {
  const { email, password } =
