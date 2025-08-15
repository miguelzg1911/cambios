import express from 'express';
import dotenv from 'dotenv';
import multer from 'multer';
import fs from 'fs';
import csvParser from 'csv-parser';
import pool from './db.js';

dotenv.config();
const app = express();
app.use(express.json());
const upload = multer({ dest: 'uploads/' });

// CRUD Endpoints

// Get all customers
app.get('/customer', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM customer');
    res.json(rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Get customer by id
app.get('/customer/:id', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM customer WHERE id_customer = ?', [req.params.id]);
    if (rows.length === 0) return res.status(404).json({ message: 'Customer not found' });
    res.json(rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Create customer
app.post('/customer', async (req, res) => {
  const { name, identification_number, email, telephone } = req.body;
  if (!name || !identification_number || !email) return res.status(400).json({ message: 'Name, identification_number and email are required' });
  try {
    const [result] = await pool.query(
      'INSERT INTO customer (name, identification_number, email, telephone) VALUES (?, ?, ?, ?)',
      [name, identification_number, email, telephone || null]
    );
    res.status(201).json({ id_customer: result.insertId, name, identification_number, email, telephone });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Update customer
app.put('/customer/:id', async (req, res) => {
  const { name, identification_number, email, telephone } = req.body;
  if (!name || !identification_number || !email) return res.status(400).json({ message: 'Name, identification_number and email are required' });
  try {
    const [result] = await pool.query(
      'UPDATE customer SET name = ?, identification_number = ?, email = ?, telephone = ? WHERE id_customer = ?',
      [name, identification_number, email, telephone || null, req.params.id]
    );
    if (result.affectedRows === 0) return res.status(404).json({ message: 'Customer not found' });
    res.json({ id_customer: req.params.id, name, identification_number, email, telephone });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Delete customer
app.delete('/customer/:id', async (req, res) => {
  try {
    const [result] = await pool.query('DELETE FROM customer WHERE id_customer = ?', [req.params.id]);
    if (result.affectedRows === 0) return res.status(404).json({ message: 'Customer not found' });
    res.json({ message: 'Customer deleted' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Upload CSV
app.post('/upload-csv', upload.single('csv'), async (req, res) => {
  if (!req.file) return res.status(400).json({ message: 'No file uploaded' });
  const results = [];
  fs.createReadStream(req.file.path)
    .pipe(csvParser({ headers: ['id_customer','name','identification_number','address','telephone','email'], skipLines: 1 }))
    .on('data', (data) => {
      data.identification_number = Number(data.identification_number);
      data.telephone = data.telephone.replace(/\D/g,'');
      results.push(data);
    })
    .on('end', async () => {
      try {
        for (const row of results) {
          await pool.query(
            'INSERT INTO customer (name, identification_number, address, telephone, email) VALUES (?, ?, ?, ?, ?)',
            [row.name, row.identification_number, row.address, row.telephone || null, row.email]
          );
        }
        fs.unlinkSync(req.file.path);
        res.json({ message: 'CSV uploaded successfully' });
      } catch (err) {
        res.status(500).json({ message: err.message });
      }
    });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));




<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Customers Dashboard</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet" />
</head>
<body class="p-4">
  <h1 class="mb-4">Customers Dashboard</h1>

  <!-- CSV Upload Form -->
  <form id="csvForm" class="mb-4">
    <div class="mb-3">
      <label for="csvFile" class="form-label">Upload CSV</label>
      <input type="file" id="csvFile" accept=".csv" class="form-control" />
    </div>
    <button type="submit" class="btn btn-success">Upload CSV</button>
  </form>

  <!-- Customer CRUD Form -->
  <form id="customerForm" class="mb-4">
    <input type="hidden" id="customerId" />
    <div class="mb-3">
      <label for="Name" class="form-label">Name</label>
      <input type="text" id="Name" class="form-control" required />
    </div>
    <div class="mb-3">
      <label for="identification_number" class="form-label">Identification Number</label>
      <input type="text" id="identification_number" class="form-control" required />
    </div>
    <div class="mb-3">
      <label for="email" class="form-label">Email</label>
      <input type="email" id="email" class="form-control" required />
    </div>
    <div class="mb-3">
      <label for="telephone" class="form-label">Telephone</label>
      <input type="text" id="telephone" class="form-control" />
    </div>
    <button type="submit" class="btn btn-primary" id="submitBtn">Add Client</button>
    <button type="button" class="btn btn-secondary" id="cancelBtn" style="display:none;">Cancel</button>
  </form>

  <table class="table table-striped" id="customersTable">
    <thead>
      <tr>
        <th>ID</th><th>Name</th><th>Identification Number</th><th>Email</th><th>Telephone</th><th>Actions</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <script src="script.js"></script>
</body>
</html>




const apiUrl = 'http://localhost:3000/customer';

const customerForm = document.getElementById('customerForm');
const customersTableBody = document.querySelector('#customersTable tbody');
const submitBtn = document.getElementById('submitBtn');
const cancelBtn = document.getElementById('cancelBtn');
const customerIdInput = document.getElementById('customerId');

const csvForm = document.getElementById('csvForm');
const csvFileInput = document.getElementById('csvFile');

// Fetch customers
async function fetchCustomers() {
  const res = await fetch(apiUrl);
  const customers = await res.json();
  customersTableBody.innerHTML = '';
  customers.forEach(c => {
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td>${c.id_customer}</td>
      <td>${c.name}</td>
      <td>${c.identification_number}</td>
      <td>${c.email}</td>
      <td>${c.telephone || ''}</td>
      <td>
        <button class="btn btn-sm btn-warning" onclick="editCustomer(${c.id_customer})">Edit</button>
        <button class="btn btn-sm btn-danger" onclick="deleteCustomer(${c.id_customer})">Delete</button>
      </td>
    `;
    customersTableBody.appendChild(tr);
  });
}

// Add or update customer
customerForm.addEventListener('submit', async e => {
  e.preventDefault();
  const customerId = customerIdInput.value;
  const data = {
    name: document.getElementById('Name').value.trim(),
    identification_number: document.getElementById('identification_number').value.trim(),
    email: document.getElementById('email').value.trim(),
    telephone: document.getElementById('telephone').value.trim() || null
  };
  if (!data.name || !data.identification_number || !data.email) {
    alert('Name, identification_number and email are required');
    return;
  }

  if (customerId) {
    await fetch(`${apiUrl}/${customerId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  } else {
    await fetch(apiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  }
  resetForm();
  fetchCustomers();
});

function resetForm() {
  customerIdInput.value = '';
  customerForm.reset();
  submitBtn.textContent = 'Add Client';
  cancelBtn.style.display = 'none';
}

cancelBtn.addEventListener('click', resetForm);

window.editCustomer = async function(id) {
  const res = await fetch(`${apiUrl}/${id}`);
  const customer = await res.json();
  customerIdInput.value = customer.id_customer;
  document.getElementById('Name').value = customer.name;
  document.getElementById('identification_number').value = customer.identification_number;
  document.getElementById('email').value = customer.email;
  document.getElementById('telephone').value = customer.telephone || '';
  submitBtn.textContent = 'Update Client';
  cancelBtn.style.display = 'inline-block';
}

window.deleteCustomer = async function(id) {
  if (confirm('Are you sure?')) {
    await fetch(`${apiUrl}/${id}`, { method: 'DELETE' });
    fetchCustomers();
  }
}

// Upload CSV
csvForm.addEventListener('submit', async e => {
  e.preventDefault();
  const file = csvFileInput.files[0];
  if (!file) return alert('Select a CSV file');
  const formData = new FormData();
  formData.append('csv', file);
  const res = await fetch('http://localhost:3000/upload-csv', { method: 'POST', body: formData });
  if (res.ok) {
    alert('CSV uploaded successfully');
    fetchCustomers();
  } else {
    const error = await res.json();
    alert('Error uploading CSV: ' + error.message);
  }
});

// Load customers on page load
fetchCustomers();




