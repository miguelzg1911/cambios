# cambios

// db.js
import mysql from "mysql2";
import dotenv from "dotenv";

dotenv.config();

// Crear un pool de conexiones
const pool = mysql.createPool({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    port: process.env.DB_PORT,
    waitForConnections: true, // Espera si no hay conexiones libres
    connectionLimit: 10,      // Número máximo de conexiones
    queueLimit: 0              // Sin límite en la cola de espera
});

// Probar la conexión al iniciar
pool.getConnection((err, connection) => {
    if (err) {
        console.error("❌ Error connecting to MySQL:", err);
        return;
    }
    console.log("✅ Connected to MySQL database (POOL)");
    connection.release(); // Liberar la conexión de prueba
});

export default pool;




// db.js
import mysql from "mysql2";
import dotenv from "dotenv";

dotenv.config();

// Crear un pool de conexiones
const pool = mysql.createPool({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    port: process.env.DB_PORT,
    waitForConnections: true, // Espera si no hay conexiones libres
    connectionLimit: 10,      // Número máximo de conexiones
    queueLimit: 0              // Sin límite en la cola de espera
});

// Probar la conexión al iniciar
pool.getConnection((err, connection) => {
    if (err) {
        console.error("❌ Error connecting to MySQL:", err);
        return;
    }
    console.log("✅ Connected to MySQL database (POOL)");
    connection.release(); // Liberar la conexión de prueba
});

export default pool;



const apiUrl = 'http://localhost:3000/clients';

const customerForm = document.getElementById('customerForm');
const customerTableBody = document.querySelector('#customerTable tbody');
const submitBtn = document.getElementById('submitBtn');
const cancelBtn = document.getElementById('cancelBtn');
const customerIdInput = document.getElementById('customerId');

// Obtener todos los clientes y mostrarlos en la tabla
async function fetchCustomers() {
  const res = await fetch(apiUrl);
  const customers = await res.json();

  customerTableBody.innerHTML = ''; // Limpia la tabla
  customers.forEach(customer => {
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td>${customer.client_id}</td>
      <td>${customer.name}</td>
      <td>${customer.identification_number}</td>
      <td>${customer.email}</td>
      <td>${customer.telephone || ''}</td>
      <td>
        <button class="btn btn-sm btn-warning" onclick="editCustomer(${customer.client_id})">Edit</button>
        <button class="btn btn-sm btn-danger" onclick="deleteCustomer(${customer.client_id})">Delete</button>
      </td>
    `;
    customerTableBody.appendChild(tr);
  });
}

// Enviar formulario (crear o actualizar)
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
    // Actualizar cliente
    const res = await fetch(`${apiUrl}/${customerId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (res.ok) resetForm();
  } else {
    // Crear cliente
    const res = await fetch(apiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (res.ok) resetForm();
  }

  fetchCustomers();
});

// Restablecer formulario
function resetForm() {
  customerIdInput.value = '';
  customerForm.reset();
  submitBtn.textContent = 'Add Customer';
  cancelBtn.style.display = 'none';
}

// Botón cancelar
cancelBtn.addEventListener('click', () => resetForm());

// Editar cliente
async function editCustomer(id) {
  const res = await fetch(`${apiUrl}/${id}`);
  if (res.ok) {
    const customer = await res.json();
    customerIdInput.value = customer.client_id;
    document.getElementById('Name').value = customer.name;
    document.getElementById('identification_number').value = customer.identification_number;
    document.getElementById('email').value = customer.email;
    document.getElementById('telephone').value = customer.telephone || '';
    submitBtn.textContent = 'Update Customer';
    cancelBtn.style.display = 'inline-block';
  }
}

// Eliminar cliente
async function deleteCustomer(id) {
  if (confirm('Are you sure you want to delete this customer?')) {
    const res = await fetch(`${apiUrl}/${id}`, { method: 'DELETE' });
    if (res.ok) fetchCustomers();
  }
}

// Cargar clientes al iniciar
fetchCustomers();


npm install express cors dotenv multer csv-parser mysql2





