const express = require("express");
const mongoose = require("mongoose");
const bodyParser = require("body-parser");

const app = express();
app.use(bodyParser.json());

// -----------------------------
// MongoDB Connection
// -----------------------------
mongoose
  .connect("mongodb://127.0.0.1:27017/bankdb", {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => console.log(" Connected to MongoDB"))
  .catch((err) => console.error(" MongoDB connection error:", err));

// -----------------------------
// Account Schema & Model
// -----------------------------
const accountSchema = new mongoose.Schema({
  name: String,
  balance: Number,
});

const Account = mongoose.model("Account", accountSchema);

// -----------------------------
// Create Account
// -----------------------------
app.post("/create-account", async (req, res) => {
  try {
    const { name, balance } = req.body;
    if (!name || balance == null) {
      return res.status(400).json({ error: "Name and balance are required" });
    }
    const account = new Account({ name, balance });
    await account.save();
    res.json({ message: "Account created successfully", account });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -----------------------------
// View All Accounts
// -----------------------------
app.get("/accounts", async (req, res) => {
  try {
    const accounts = await Account.find();
    res.json(accounts);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// -----------------------------
// Transfer Money (Core Logic)
// -----------------------------
app.post("/transfer", async (req, res) => {
  try {
    const { senderName, receiverName, amount } = req.body;

    if (!senderName || !receiverName || !amount || amount <= 0) {
      return res.status(400).json({ error: "Invalid transfer request" });
    }

    const sender = await Account.findOne({ name: senderName });
    const receiver = await Account.findOne({ name: receiverName });

    if (!sender) return res.status(404).json({ error: "Sender not found" });
    if (!receiver) return res.status(404).json({ error: "Receiver not found" });

    if (sender.balance < amount) {
      return res.status(400).json({ error: "Insufficient balance" });
    }

    // Deduct and update balances sequentially
    sender.balance -= amount;
    receiver.balance += amount;

    await sender.save();
    await receiver.save();

    res.json({
      message: `Transferred $${amount} from ${senderName} to ${receiverName}`,
      senderBalance: sender.balance,
      receiverBalance: receiver.balance,
    });
  } catch (err) {
    res.status(500).json({ error: "Transfer failed: " + err.message });
  }
});

// -----------------------------
// Start Server
// -----------------------------
const PORT = 3000;
app.listen(PORT, () => {
  console.log(` Server running on http://localhost:${PORT}`);
});
