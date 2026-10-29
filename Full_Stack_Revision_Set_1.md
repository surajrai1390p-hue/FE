# Full Stack Development – Practical Revision (Set 1)
**Clean, copy-paste-ready code. Each question: File → Setup → Code → Run/Test → Key points.**

## Before you start (applies to all)
1. Install Node.js.
2. Open the terminal **inside your project folder**.
3. Express questions: `npm install express`
4. Sequelize + MySQL questions: `npm install sequelize mysql2`
5. Replace `your_password` with your real MySQL password.
6. MySQL Server must be running and the database must already be created.
7. React files go inside the `src` folder of your React project.
8. Never run two servers on the same port at the same time.

---

---

# REVISION SET 1
---

## Q1 (a). Express health route, error handling and 404 (5 marks)
**File:** `server.js`

**Setup**
```bash
npm init -y
npm install express
```

**Code**
```javascript
const express = require("express");
const app = express();

// 1. Normal routes
app.get("/health", (req, res) => {
    res.json({ status: "OK", message: "Service is running" });
});

app.get("/test-error", (req, res, next) => {
    next(new Error("Something went wrong"));
});

// 2. 404 handler (must come AFTER all routes)
app.use((req, res) => {
    res.status(404).json({ message: "Route not found" });
});

// 3. Error handler (must have 4 parameters, comes LAST)
app.use((err, req, res, next) => {
    res.status(500).json({ message: err.message });
});

// 4. Start server
app.listen(3000, () => {
    console.log("Server running on port 3000");
});
```

**Run:** `node server.js`

**Test**
| URL | Response | Status |
|---|---|---|
| `GET /health` | `{"status":"OK","message":"Service is running"}` | 200 |
| `GET /test-error` | `{"message":"Something went wrong"}` | 500 |
| `GET /unknown` | `{"message":"Route not found"}` | 404 |

**Key points**
- `app.get()` creates a GET route; `res.json()` sends JSON.
- `next(new Error(...))` sends the error to the error middleware.
- **Order to remember:** routes → 404 middleware → error middleware → listen.

---

## Q1 (b). Read port and app mode from environment variables (5 marks)
**File:** `config.js`

**Code**
```javascript
const PORT = process.env.PORT || 3000;
const APP_MODE = process.env.APP_MODE || "development";

console.log("Port:", PORT);
console.log("Application Mode:", APP_MODE);
```

**Run with defaults**
```bash
node config.js
```
Output:
```
Port: 3000
Application Mode: development
```

**Run with custom values**

Windows PowerShell:
```powershell
$env:PORT=5000; $env:APP_MODE="production"; node config.js
```
Windows CMD:
```cmd
set PORT=5000 && set APP_MODE=production && node config.js
```
Mac / Linux:
```bash
PORT=5000 APP_MODE=production node config.js
```
Output:
```
Port: 5000
Application Mode: production
```

**Key points**
- `process.env.NAME` reads an environment variable.
- `||` gives a default value when the variable is missing.

---

## Q2 (a). Delivery estimates with Promise.all() (10 marks)
**File:** `delivery.js`

**Code**
```javascript
function getDeliveryEstimate(days) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (days < 0) {
                reject(new Error("Days cannot be negative"));
                return;
            }

            const date = new Date();
            date.setDate(date.getDate() + days);
            resolve(date.toDateString());
        }, 1000);
    });
}

Promise.all([
    getDeliveryEstimate(2),
    getDeliveryEstimate(4),
    getDeliveryEstimate(6)
])
    .then(results => {
        console.log("Delivery estimates:");
        results.forEach((date, index) => {
            console.log("Order " + (index + 1) + ":", date);
        });
    })
    .catch(error => {
        console.log("Error:", error.message);
    });
```

**Run:** `node delivery.js`

**Key points**
- `resolve()` = success, `reject()` = failure.
- `Promise.all()` waits for **all** promises to succeed.
- If **any** promise fails, `.catch()` runs.
- All three run at the same time (not one after another).

---

## Q2 (b). React password checker (7 marks)
**File:** `src/PasswordChecker.jsx`

**Code**
```jsx
import { useState } from "react";

function PasswordChecker() {
    const [password, setPassword] = useState("");

    const hasLength = password.length >= 8;
    const hasUppercase = /[A-Z]/.test(password);
    const hasNumber = /[0-9]/.test(password);

    return (
        <div>
            <h2>Password Checker</h2>

            <input
                type="password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                placeholder="Enter password"
            />

            <p>{hasLength ? "✓" : "✗"} At least 8 characters</p>
            <p>{hasUppercase ? "✓" : "✗"} One uppercase letter</p>
            <p>{hasNumber ? "✓" : "✗"} One number</p>

            {hasLength && hasUppercase && hasNumber && (
                <p>Password meets all requirements!</p>
            )}
        </div>
    );
}

export default PasswordChecker;
```

**File:** `src/App.jsx`
```jsx
import PasswordChecker from "./PasswordChecker";

function App() {
    return <PasswordChecker />;
}

export default App;
```

**Key points**
- `useState` stores the password.
- `value` + `onChange` = controlled input.
- `/[A-Z]/` checks uppercase, `/[0-9]/` checks a number.
- Success message shows only when all 3 conditions are true.

---

## Q2 (c). Sequelize workshop model, bulkCreate and pagination (7 marks)
**File:** `app.js`

**Setup**

In MySQL:
```sql
CREATE DATABASE workshop_db;
```
In terminal:
```bash
npm init -y
npm install sequelize mysql2
```

**Code**
```javascript
const { Sequelize, DataTypes } = require("sequelize");

const sequelize = new Sequelize("workshop_db", "root", "your_password", {
    host: "localhost",
    dialect: "mysql"
});

const Workshop = sequelize.define("Workshop", {
    title: { type: DataTypes.STRING, allowNull: false },
    capacity: DataTypes.INTEGER,
    fee: DataTypes.FLOAT
});

async function main() {
    try {
        await sequelize.authenticate();
        await sequelize.sync();

        // Insert 5 sample rows
        await Workshop.bulkCreate([
            { title: "React", capacity: 30, fee: 500 },
            { title: "Node.js", capacity: 25, fee: 700 },
            { title: "Python", capacity: 40, fee: 400 },
            { title: "Cyber Security", capacity: 20, fee: 900 },
            { title: "Database", capacity: 35, fee: 600 }
        ]);

        // Pagination: first page, 2 rows, highest fee first
        const result = await Workshop.findAndCountAll({
            limit: 2,
            offset: 0,
            order: [["fee", "DESC"]]
        });

        console.log("Total matching workshops:", result.count);
        console.log("Current page:");
        console.log(result.rows.map((w) => w.toJSON()));
    } catch (error) {
        console.log("Error:", error.message);
    } finally {
        await sequelize.close();
    }
}

main();
```

**Run:** `node app.js`

**Key points**
- `sequelize.define()` creates the model.
- `bulkCreate()` inserts many rows at once.
- `findAndCountAll()` returns `count` (total rows) and `rows` (current page).
- `limit` = rows per page, `offset` = rows to skip, `order fee DESC` = highest fee first.
- ⚠ Running the file again inserts the same 5 rows again. Clear the table if needed.

---

## Q2 (d). Express API for total workshop capacity + React component (4 marks)

### Backend – `server.js` (uses same `workshop_db` as Q2(c))
```javascript
const express = require("express");
const { Sequelize, DataTypes, fn, col } = require("sequelize");

const app = express();

const sequelize = new Sequelize("workshop_db", "root", "your_password", {
    host: "localhost",
    dialect: "mysql"
});

const Workshop = sequelize.define("Workshop", {
    title: DataTypes.STRING,
    capacity: DataTypes.INTEGER,
    fee: DataTypes.FLOAT
});

app.get("/api/workshops/capacity", async (req, res) => {
    try {
        const result = await Workshop.findOne({
            attributes: [[fn("SUM", col("capacity")), "totalCapacity"]],
            raw: true
        });

        res.json({ totalCapacity: Number(result.totalCapacity || 0) });
    } catch (error) {
        res.status(500).json({ message: "Unable to calculate capacity" });
    }
});

async function startServer() {
    try {
        await sequelize.authenticate();
        await sequelize.sync();

        app.listen(3000, () => {
            console.log("Server running on port 3000");
        });
    } catch (error) {
        console.log("Database connection failed:", error.message);
    }
}

startServer();
```

**Test:** `GET http://localhost:3000/api/workshops/capacity`
Response (for the 5 sample workshops): `{"totalCapacity":150}`

### Frontend – `src/Capacity.jsx`
```jsx
import { useEffect, useState } from "react";

function Capacity() {
    const [total, setTotal] = useState(0);

    useEffect(() => {
        fetch("http://localhost:3000/api/workshops/capacity")
            .then((res) => {
                if (!res.ok) throw new Error("Request failed");
                return res.json();
            })
            .then((data) => setTotal(data.totalCapacity))
            .catch((error) => console.log(error.message));
    }, []);

    return (
        <div>
            <h2>Total Workshop Capacity</h2>
            <p>{total}</p>
        </div>
    );
}

export default Capacity;
```

**Key points**
- `SUM(capacity)` adds all capacities.
- React `fetch()` inside `useEffect(..., [])` runs once when the component loads.
- `setTotal()` updates the screen.
- If frontend and backend run on different ports, you may need CORS (`npm install cors`, then `app.use(require("cors")())`) or a Vite proxy.

---

## Q3 (a). React confirmation dialog using createPortal() (5 marks)
**File:** `src/App.jsx`

**Code**
```jsx
import { useState } from "react";
import { createPortal } from "react-dom";

function ConfirmDialog({ onConfirm, onCancel }) {
    return createPortal(
        <div>
            <h3>Are you sure you want to delete this item?</h3>
            <button onClick={onConfirm}>Confirm</button>
            <button onClick={onCancel}>Cancel</button>
        </div>,
        document.body
    );
}

function App() {
    const [showDialog, setShowDialog] = useState(false);
    const [message, setMessage] = useState("");

    function confirmDelete() {
        setMessage("Item deleted");
        setShowDialog(false);
    }

    function cancelDelete() {
        setShowDialog(false);
    }

    return (
        <div>
            <h2>Delete Example</h2>
            <button onClick={() => setShowDialog(true)}>Delete item</button>

            {showDialog && (
                <ConfirmDialog
                    onConfirm={confirmDelete}
                    onCancel={cancelDelete}
                />
            )}

            <p>{message}</p>
        </div>
    );
}

export default App;
```

**Key points**
- `useState` controls whether the dialog is visible.
- `createPortal(content, document.body)` renders the dialog directly inside `<body>`.
- Confirm → shows message and closes. Cancel → only closes.

---

## Q3 (b). Correct React email form (5 marks)
**File:** `src/EmailForm.jsx`

**Code**
```jsx
import { useState } from "react";

function EmailForm() {
    const [email, setEmail] = useState("");
    const [message, setMessage] = useState("");

    function submitForm(e) {
        e.preventDefault();               // stops page reload
        setMessage("Email saved: " + email);
    }

    return (
        <div>
            <h2>Email Form</h2>

            <form onSubmit={submitForm}>
                <input
                    type="email"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                    placeholder="Enter email"
                    required
                />
                <button type="submit">Save</button>
            </form>

            <p>{message}</p>
        </div>
    );
}

export default EmailForm;
```

**Key points**
- The original code failed because it did not stop the default form submit.
- `e.preventDefault()` prevents the page reload.
- `onChange` keeps the email state updated.

---

---

# QUICK REVISION – LOGIC TO REMEMBER

## Set 1
| Q | Flow |
|---|---|
| 1(a) | Routes → 404 middleware → error middleware → listen |
| 1(b) | `process.env.NAME \|\| default` |
| 2(a) | Promise → resolve/reject → `Promise.all` → then/catch |
| 2(b) | `useState` → controlled input → validation conditions |
| 2(c) | define model → `bulkCreate` → `findAndCountAll` → count/rows |
| 2(d) | `SUM` aggregate → Express JSON API → React `fetch` + `useEffect` |
| 3(a) | `useState` controls visibility → `createPortal` renders dialog |
| 3(b) | `onSubmit` → `preventDefault` → update message state |

**END**
