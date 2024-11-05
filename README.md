# Emotoradnpm 
npm install express sqlite3 body-parser

const express = require('express');
const bodyParser = require('body-parser');
const sqlite3 = require('sqlite3').verbose();
const app = express();
app.use(bodyParser.json());

// Initialize Database
const db = new sqlite3.Database(':memory:'); // Using an in-memory DB for simplicity

// Create contacts table based on provided schema
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS contacts (
    id INTEGER PRIMARY KEY,
    phoneNumber TEXT,
    email TEXT,
    linkedId INTEGER,
    linkPrecedence TEXT CHECK(linkPrecedence IN ('primary', 'secondary')),
    createdAt TEXT,
    updatedAt TEXT,
    deletedAt TEXT
  )`);
});

// Helper function to get contact by email or phone number
const findContact = async (email, phoneNumber) => {
  return new Promise((resolve, reject) => {
    db.all(
      `SELECT * FROM contacts WHERE email = ? OR phoneNumber = ?`,
      [email, phoneNumber],
      (err, rows) => (err ? reject(err) : resolve(rows))
    );
  });
};

// Helper function to insert a new contact
const insertContact = async (contact) => {
  return new Promise((resolve, reject) => {
    const now = new Date().toISOString();
    db.run(
      `INSERT INTO contacts (phoneNumber, email, linkedId, linkPrecedence, createdAt, updatedAt) VALUES (?, ?, ?, ?, ?, ?)`,
      [contact.phoneNumber, contact.email, contact.linkedId, contact.linkPrecedence, now, now],
      function (err) {
        if (err) reject(err);
        resolve(this.lastID);
      }
    );
  });
};

// /identify endpoint to handle requests
app.post('/identify', async (req, res) => {
  const { email, phoneNumber } = req.body;

  if (!email && !phoneNumber) {
    return res.status(400).json({ error: 'Email or phone number required.' });
  }

  try {
    // Find existing contacts
    let contacts = await findContact(email, phoneNumber);

    // If no matching contacts, create a new primary contact
    if (contacts.length === 0) {
      const newContactId = await insertContact({
        phoneNumber,
        email,
        linkedId: null,
        linkPrecedence: 'primary',
      });
      return res.status(200).json({
        primaryContactId: newContactId,
        emails: [email],
        phoneNumbers: [phoneNumber],
        secondaryContactIds: [],
      });
    }

    // Process existing contacts and consolidate data
    let primaryContact = contacts.find((c) => c.linkPrecedence === 'primary') || contacts[0];
    let primaryContactId = primaryContact.id;
    let emails = [...new Set(contacts.map((c) => c.email).filter(Boolean))];
    let phoneNumbers = [...new Set(contacts.map((c) => c.phoneNumber).filter(Boolean))];
    let secondaryContactIds = contacts.filter((c) => c.linkPrecedence === 'secondary').map((c) => c.id);

    // Check if new entry is needed
    if (!contacts.some((c) => c.email === email || c.phoneNumber === phoneNumber)) {
      const newSecondaryId = await insertContact({
        phoneNumber,
        email,
        linkedId: primaryContactId,
        linkPrecedence: 'secondary',
      });
      secondaryContactIds.push(newSecondaryId);
      if (email) emails.push(email);
      if (phoneNumber) phoneNumbers.push(phoneNumber);
    }

    // Respond with consolidated data
    res.status(200).json({
      primaryContactId,
      emails,
      phoneNumbers,
      secondaryContactIds,
    });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal Server Error' });
  }
});

// Start the server
app.listen(3000, () => {
  console.log('Server is running on port 3000');
});
