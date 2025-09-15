// app.js
const express = require("express");
const bodyParser = require("body-parser");
const jwt = require("jsonwebtoken");
const fs = require("fs");
const cors = require("cors");
const axios = require("axios");
const cron = require("node-cron");
require("dotenv").config();

const app = express();
app.use(bodyParser.json());
app.use(cors());

// ---------------- Config ----------------
const PORT = process.env.PORT || 4000;
const JWT_SECRET = process.env.JWT_SECRET || "supersecret";
const dbFile = "./db.json";

function loadDB() {
  if (!fs.existsSync(dbFile)) {
    return { users: [], deposits: [], withdrawals: [], transactions: [] };
  }
  return JSON.parse(fs.readFileSync(dbFile));
}
function saveDB(data) {
  fs.writeFileSync(dbFile, JSON.stringify(data, null, 2));
}
let db = loadDB();

// ---------------- Helpers ----------------
function generateId(prefix) {
  return prefix + "_" + Math.random().toString(36).substr(2, 9);
}
function auth(req, res, next) {
  const token = req.headers["authorization"];
  if (!token) return res.status(401).json({ error: "No token" });
  try {
    const decoded = jwt.verify(token.split(" ")[1], JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
}
function sendSMS(to, message) {
  if (process.env.AT_API_KEY && process.env.AT_USERNAME) {
    console.log(`(Real SMS via Africa's Talking would go to ${to}: ${message})`);
  } else {
    console.log(`SMS (not sent) -> ${to}: ${message}`);
  }
}
function mpBase() {
  return process.env.MPESA_ENV === "production"
    ? "https://api.safaricom.co.ke"
    : "https://sandbox.safaricom.co.ke";
}
async function getMpesaToken() {
  const url = `${mpBase()}/oauth/v1/generate?grant_type=client_credentials`;
  const auth = Buffer.from(
    `${process.env.MPESA_CONSUMER_KEY}:${process.env.MPESA_CONSUMER_SECRET}`
  ).toString("base64");
  const { data } = await axios.get(url, {
    headers: { Authorization: `Basic ${auth}` },
  });
  return data.access_token;
}

// ---------------- Auth ----------------
app.post("/signup", (req, res) => {
  const { phone, password, referral } = req.body;
  if (db.users.find((u) => u.phone === phone)) {
    return res.status(400).json({ error: "User exists" });
  }
  const user = {
    id: generateId("u"),
    p
