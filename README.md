# SammyJay02-Site
Airtime and Data vending
sammyjay-site/server.sammyjay-site
├── backend/
│   ├── .envPORT=4242
MONGO_URI=mongodb://localhost:27017/sammyjay
STRIPE_SECRET_KEY=sk_test_your_secret_key
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret
JWT_SECRET=your_super_secret_key
│   ├── package.json{
  "name": "sammyjay-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "mongoose": "^7.0.0",
    "stripe": "^11.0.0"
  }
}
│   ├── serequire('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const Order = require('./models/Order');
const User = require('./models/User');

const app = express();
const port = process.env.PORT || 4242;
const JWT_SECRET = process.env.JWT_SECRET;

app.use(cors());
app.use(express.json());
app.use(express.static('../public'));

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error(err));

// AUTH ROUTES
app.post('/signup', async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password)
    return res.status(400).json({ error: 'Email and password required' });

  const existing = await User.findOne({ email });
  if (existing)
    return res.status(409).json({ error: 'Email already registered' });

  const hash = await bcrypt.hash(password, 10);
  const user = await new User({ email, passwordHash: hash }).save();
  const token = jwt.sign({ userId: user._id, role: user.role }, JWT_SECRET, { expiresIn: '7d' });
  res.json({ token });
});

app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user)
    return res.status(401).json({ error: 'Invalid credentials' });

  const valid = await bcrypt.compare(password, user.passwordHash);
  if (!valid)
    return res.status(401).json({ error: 'Invalid credentials' });

  const token = jwt.sign({ userId: user._id, role: user.role }, JWT_SECRET, { expiresIn: '7d' });
  res.json({ token });
});

// JWT MIDDLEWARE
function authMiddleware(role) {
  return (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'Unauthorized' });

    try {
      const payload = jwt.verify(token, JWT_SECRET);
      if (role && payload.role !== role) return res.status(403).json({ error: 'Forbidden' });
      req.user = payload;
      next();
    } catch {
      res.status(401).json({ error: 'Invalid token' });
    }
  };
}

// PAYMENT INTENT
app.post('/create-payment-intent', authMiddleware('customer'), async (req, res) => {
  try {
    const { network, phone, amount } = req.body;
    const order = await new Order({ network, phone, amount }).save();
    const paymentIntent = await stripe.paymentIntents.create({
      amount: amount * 100,
      currency: 'ngn',
      metadata: { orderId: order._id.toString() },
    });
    order.stripePaymentIntentId = paymentIntent.id;
    await order.save();
    res.json({ clientSecret: paymentIntent.client_secret });
  } catch (err) {
    console.error(err);
    res.status(400).json({ error: err.message });
  }
});

// STRIPE WEBHOOK
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['stripe-signature'];
  let event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    console.error('Webhook Error:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if (event.type === 'payment_intent.succeeded') {
    const pi = event.data.object;
    Order.findOneAndUpdate({ stripePaymentIntentId: pi.id }, { paid: true })
      .then(order => console.log('Order marked as paid:', order?._id));
  }

  res.json({ received: true });
});

// ADMIN ROUTE
app.get('/admin/orders', authMiddleware('admin'), async (req, res) => {
  const orders = await Order.find().sort({ createdAt: -1 });
  res.json(orders);
});

app.listen(port, () => console.log(`Backend running on http://localhost:${port}`));rver.js
│   └── models/
│       ├── Order.const mongoose = require('mongoose');

const OrderSchema = new mongoose.Schema({
  network: String,
  phone: String,
  amount: Number,
  stripePaymentIntentId: String,
  paid: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Order', OrderSchema);js
│       └──const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  email: { type: String, unique: true, required: true },
  passwordHash: { type: String, required: true },
  role: { type: String, enum: ['customer', 'admin'], default: 'customer' },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', UserSchema); User.js
└── public/
    ├── <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Sammy Jay | Airtime & Data</title>
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <header class="header">Sammy Jay</header>
  <nav class="nav">
    <a href="index.html">Home</a>
    <a href="signup.html">Signup</a>
    <a href="login.html">Login</a>
    <a href="admin.html">Admin</a>
  </nav>

  <section class="hero container">
    <h1>Fast & Affordable Airtime & Data</h1>
    <p>Buy for all networks instantly!</p>
  </section>

  <section class="services container">
    <div class="card">
      <h3>📱 Airtime</h3>
      <p>Top up MTN, Airtel, Glo, 9mobile quickly and securely.</p>
    </div>
    <div class="card">
      <h3>🌐 Data</h3>
      <p>Affordable data plans for all networks. Stay connected anywhere.</p>
    </div>
  </section>

  <section class="container">
    <h2>Purchase Airtime or Data</h2>
    <form id="purchase-form" class="card">
      <div class="form-group">
        <label>Network:</label>
        <select name="network" required>
          <option>MTN</option><option>Airtel</option><option>Glo</option><option>9mobile</option>
        </select>
      </div>
      <div class="form-group">
        <label>Phone Number:</label>
        <input type="tel" name="phone" placeholder="08012345678" pattern="0[0-9]{10}" required>
      </div>
      <div class="form-group">
        <label>Amount (₦):</label>
        <input type="number" name="amount" placeholder="e.g. 500" min="100" required>
      </div>
      <button type="submit">Proceed to Payment</button>
    </form>
  </section>

  <section class="whatsapp container">
    <a href="https://wa.me/2349121658071" target="_blank">💬 Contact on WhatsApp</a>
  </section>

  <footer class="footer">&copy; 2025 Sammy Jay. All rights reserved.</footer>
  <script src="https://js.stripe.com/v3/"></script>
  <script src="js/app.js"></script>
</body>
</html>index.html
    ├── <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Signup | Sammy Jay</title>
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <header class="header">Sammy Jay</header>
  <nav class="nav">
    <a href="index.html">Home</a>
    <a href="login.html">Login</a>
  </nav>

  <div class="container">
    <div class="card" style="max-width: 400px; margin: auto;">
      <h2>Create Account</h2>
      <form id="signup-form">
        <div class="form-group">
          <label>Email</label>
          <input name="email" type="email" required>
        </div>
        <div class="form-group">
          <label>Password</label>
          <input name="password" type="password" required>
        </div>
        <button type="submit">Signup</button>
      </form>
    </div>
  </div>

  <footer class="footer">&copy; 2025 Sammy Jay.</footer>

  <script>
    document.getElementById('signup-form').addEventListener('submit', async e => {
      e.preventDefault();
      const form = e.target;
      const res = await fetch('/signup', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email: formsignup.html
    ├── <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login | Sammy Jay</title>
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <header class="header">Sammy Jay</header>
  <nav class="nav">
    <a href="index.html">Home</a>
    <a href="signup.html">Signup</a>
  </nav>

  <div class="container">
    <div class="card" style="max-width: 400px; margin: auto;">
      <h2>Login</h2>
      <form id="login-form">
        <div class="form-group">
          <label>Email</label>
          <input name="email" type="email" required>
        </div>
        <div class="form-group">
          <label>Password</label>
          <input name="password" type="password" required>
        </div>
        <button type="submit">Login</button>
      </form>
    </div>
  </div>

  <footer class="footer">&copy; 2025 Sammy Jay.</footer>

  <script>
    document.getElementById('login-form').addEventListener('submit', async e => {
      e.preventDefault();
      const form = e.target;
      const res = await fetch('/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email: form.email.value,
          password: form.password.value
        })
      });
      const data = await res.json();
      if (res.ok) {
        localStorage.setItem('token', data.token);
        window.location.href = 'index.html';
      } else {
        alert(data.error);
      }
    });
  </script>
</body>
</html>login.html
    ├── <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Admin Panel | Sammy Jay</title>
  <link rel="stylesheet" href="css/styles.css" />
</head>
<body>
  <header class="header">Admin Panel</header>
  <nav class="nav">
    <a href="index.html">Home</a>
    <a href="login.html">Logout</a>
  </nav>

  <div class="container">
    <h2>All Orders</h2>
    <div id="orders" class="services"></div>
  </div>

  <footer class="footer">&copy; 2025 Sammy Jay.</footer>

  <script>
    (async () => {
      const token = localStorage.getItem('token');
      if (!token) return window.location.href = 'login.html';

      const res = await fetch('/admin/orders', {
        headers: {
          Authorization: 'Bearer ' + token
        }
      });

      if (!res.ok) return alert('Access denied');

      const orders = await res.json();
      const container = document.getElementById('orders');

      container.innerHTML = orders.map(order => `
        <div class="card">
          <h3>${order.network}</h3>
          <p>Phone: ${order.phone}</p>
          <p>Amount: ₦${order.amount}</p>
          <p>Status: <strong>${order.paid ? '✅ Paid' : '❌ Pending'}</strong></p>
          <p>Date: ${new Date(order.createdAt).toLocaleString()}</p>
        </div>
      `).join('');
    })();
  </script>
</body>
</html>admin.html
    ├── <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Thank You | Sammy Jay</title>
  <link rel="stylesheet" href="css/styles.css" />
</head>
<body>
  <header class="header">Sammy Jay</header>
  <nav class="nav">
    <a href="index.html">Home</a>
  </nav>

  <div class="container">
    <div class="card" style="text-align: center;">
      <h2>✅ Thank you for your purchase!</h2>
      <p>Your airtime/data transaction was received. You'll get a confirmation shortly.</p>
      <a href="https://wa.me/2349121658071" target="_blank">
        <button style="margin-top: 15px;">💬 Need help? Chat on WhatsApp</button>
      </a>
    </div>
  </div>

  <footer class="footer">&copy; 2025 Sammy Jay.</footer>
</body>
</html>checkout.html
    ├── css/
    │   └── body, h1, h2, h3, p, a, input, select, button {
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}
.container { max-width: 1000px; margin: auto; padding: 20px; }
.header, footer { background: #0d47a1; color: #fff; text-align: center; padding: 15px; }
.nav { background: #1565c0; padding: 10px; text-align: center; }
.nav a { margin: 0 15px; color: #fff; font-weight: bold; transition: color .3s; }
.nav a:hover { color: #ffd600; }
.hero { background: #e3f2fd; padding: 40px 20px; text-align: center; }
.card { background: #fff; padding: 20px; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); transition: transform .2s; }
.card:hover { transform: scale(1.02); }
.services { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 20px; margin: 40px 0; }
.whatsapp { text-align: center; margin: 40px 0; }
.whatsapp a { background: #25D366; color: #fff; padding: 15px 25px; border-radius: 50px; font-size: 1.1rem; display: inline-block; }
.form-group { margin-bottom: 15px; }
input, select { width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 5px; }
button { background: #0d47a1; color: #fff; border: none; padding: 12px 20px; border-radius: 5px; cursor: pointer; transition: background .3s; }
button:hover { background: #1565c0; }styles.css
    └── js/
        └── const form = document.getElementById('purchase-form');
if (form) form.addEventListener('submit', async e => {
  e.preventDefault();
  const network = form.network.value;
  const phone = form.phone.value;
  const amount = form.amount.value;
  const token = localStorage.getItem('token');

  if (!token) return alert('Please login first.');

  const res = await fetch('/create-payment-intent', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + token
    },
    body: JSON.stringify({ network, phone, amount })
  });

  const { clientSecret } = await res.json();
  const stripe = Stripe('YOUR_STRIPE_PUBLISHABLE_KEY');
  const result = await stripe.confirmCardPayment(clientSecret, {
    payment_method: { card: {} }
  });

  if (result.error) {
    alert(result.error.message);
  } else {
    window.location.href = 'checkout.html';
  }
});app.js