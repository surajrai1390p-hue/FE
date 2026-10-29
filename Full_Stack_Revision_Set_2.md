# Full Stack Development – Practical Revision (Set 2)
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

# REVISION SET 2
---

## Q1 (a). Express preferences service (5 marks)
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

app.use(express.json());   // needed to read JSON body

let preference = { theme: "light" };

// Read
app.get("/preferences", (req, res) => {
    res.json(preference);
});

// Update (with validation)
app.put("/preferences", (req, res) => {
    const { theme } = req.body;

    if (theme !== "light" && theme !== "dark") {
        return res.status(400).json({ message: "Theme must be light or dark" });
    }

    preference.theme = theme;
    res.json(preference);
});

app.listen(3000, () => {
    console.log("Server running on port 3000");
});
```

**Run:** `node server.js`

**Test (Postman)**
| Request | Body (raw → JSON) | Response | Status |
|---|---|---|---|
| `GET /preferences` | – | `{"theme":"light"}` | 200 |
| `PUT /preferences` | `{"theme":"dark"}` | `{"theme":"dark"}` | 200 |
| `PUT /preferences` | `{"theme":"red"}` | `{"message":"Theme must be light or dark"}` | 400 |

**Key points**
- `express.json()` parses JSON bodies.
- GET reads, PUT validates then updates.
- 400 = invalid input.

---

## Q1 (b). util.inspect() on a course object (5 marks)
**File:** `course.js`

**Code**
```javascript
const util = require("util");

const course = {
    code: "CSA-301",
    title: "Full Stack Development",
    students: ["Akash", "Rahul", "Priya"]
};

console.log(util.inspect(course, { depth: 2, colors: false }));
```

**Run:** `node course.js`

**Output**
```
{
  code: 'CSA-301',
  title: 'Full Stack Development',
  students: [ 'Akash', 'Rahul', 'Priya' ]
}
```

**Key points**
- `util` is a built-in Node.js module (no install needed).
- `util.inspect()` gives a readable view of an object (good for debugging).
- `depth` = how deep nested objects are shown; `colors: false` = no coloured output.

---

## Q2 (a). Service checks using Promise.allSettled() (10 marks)
**File:** `services.js`

**Code**
```javascript
function checkService(name, success) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (success) {
                resolve(name + " is working");
            } else {
                reject(new Error(name + " failed"));
            }
        }, 1000);
    });
}

async function checkAllServices() {
    const services = ["Database", "Mail Service", "File Storage"];

    const results = await Promise.allSettled([
        checkService("Database", true),
        checkService("Mail Service", false),
        checkService("File Storage", true)
    ]);

    results.forEach((result, index) => {
        if (result.status === "fulfilled") {
            console.log(services[index] + ": SUCCESS");
            console.log(result.value);
        } else {
            console.log(services[index] + ": FAILED");
            console.log(result.reason.message);
        }
    });
}

checkAllServices();
```

**Run:** `node services.js`

**Output**
```
Database: SUCCESS
Database is working
Mail Service: FAILED
Mail Service failed
File Storage: SUCCESS
File Storage is working
```

**Key points**
- `Promise.allSettled()` waits for **every** promise, whether it succeeds or fails.
- `status === "fulfilled"` → use `result.value`.
- `status === "rejected"` → use `result.reason`.
- Difference from `Promise.all()`: `allSettled` never stops early.

---

## Q2 (b). React task progress component (7 marks)
**File:** `src/TaskProgress.jsx`

**Code**
```jsx
function TaskProgress({ tasks = [] }) {
    const completed = tasks.filter((task) => task.completed).length;

    const percentage =
        tasks.length === 0 ? 0 : Math.round((completed / tasks.length) * 100);

    if (tasks.length === 0) {
        return (
            <div>
                <h2>Task Progress</h2>
                <p>No tasks available</p>
                <progress value={0} max="100" />
                <p>0% completed</p>
            </div>
        );
    }

    return (
        <div>
            <h2>Task Progress</h2>

            {tasks.map((task, index) => (
                <p key={index}>
                    {task.title} - {task.completed ? "Completed" : "Pending"}
                </p>
            ))}

            <progress value={percentage} max="100" />
            <p>{percentage}% completed</p>
        </div>
    );
}

export default TaskProgress;
```

**File:** `src/App.jsx`
```jsx
import TaskProgress from "./TaskProgress";

function App() {
    const tasks = [
        { title: "Learn React", completed: true },
        { title: "Learn Node.js", completed: true },
        { title: "Learn Sequelize", completed: false }
    ];

    return <TaskProgress tasks={tasks} />;
}

export default App;
```

**Expected output**
```
Learn React - Completed
Learn Node.js - Completed
Learn Sequelize - Pending
67% completed
```

**Key points**
- `filter()` counts completed tasks.
- Percentage = completed ÷ total × 100.
- `map()` shows each task; `<progress>` shows the bar.
- `tasks = []` default prevents errors if the prop is missing.

---

## Q2 (c). Transfer funds with a Sequelize transaction (7 marks)

**Setup**

In MySQL:
```sql
CREATE DATABASE bank_db;
```
In terminal:
```bash
npm install sequelize mysql2
```

### Step 1 – Create the accounts once: `seed.js`
```javascript
const { Sequelize, DataTypes } = require("sequelize");

const sequelize = new Sequelize("bank_db", "root", "your_password", {
    host: "localhost",
    dialect: "mysql"
});

const Account = sequelize.define("Account", {
    holderName: DataTypes.STRING,
    balance: { type: DataTypes.DECIMAL(12, 2), allowNull: false, defaultValue: 0 }
});

async function seed() {
    await sequelize.sync();
    await Account.bulkCreate([
        { holderName: "Account One", balance: 2000 },
        { holderName: "Account Two", balance: 1000 }
    ]);
    console.log("Accounts created");
    await sequelize.close();
}

seed();
```
Run **only once:** `node seed.js`

### Step 2 – The transfer: `transfer.js`
```javascript
const { Sequelize, DataTypes } = require("sequelize");

const sequelize = new Sequelize("bank_db", "root", "your_password", {
    host: "localhost",
    dialect: "mysql"
});

const Account = sequelize.define("Account", {
    holderName: DataTypes.STRING,
    balance: { type: DataTypes.DECIMAL(12, 2), allowNull: false, defaultValue: 0 }
});

const TransferLog = sequelize.define("TransferLog", {
    fromAccountId: DataTypes.INTEGER,
    toAccountId: DataTypes.INTEGER,
    amount: DataTypes.DECIMAL(12, 2)
});

async function transfer(fromId, toId, amount) {
    const t = await sequelize.transaction();

    try {
        if (!Number.isFinite(amount) || amount <= 0) {
            throw new Error("Amount must be greater than zero");
        }

        const sender = await Account.findByPk(fromId, {
            transaction: t,
            lock: t.LOCK.UPDATE
        });
        const receiver = await Account.findByPk(toId, {
            transaction: t,
            lock: t.LOCK.UPDATE
        });

        if (!sender || !receiver) {
            throw new Error("Account not found");
        }
        if (Number(sender.balance) < amount) {
            throw new Error("Insufficient balance");
        }

        await sender.decrement("balance", { by: amount, transaction: t });   // debit
        await receiver.increment("balance", { by: amount, transaction: t }); // credit

        await TransferLog.create(
            { fromAccountId: fromId, toAccountId: toId, amount },
            { transaction: t }
        );

        await t.commit();
        console.log("Transfer successful");
    } catch (error) {
        await t.rollback();
        console.log("Transfer failed:", error.message);
    }
}

async function main() {
    try {
        await sequelize.authenticate();
        await sequelize.sync();

        // Account 1 sends 500 to Account 2
        await transfer(1, 2, 500);
    } catch (error) {
        console.log("Database error:", error.message);
    } finally {
        await sequelize.close();
    }
}

main();
```

**Run:** `node transfer.js`
**Result:** Account 1 = 1500, Account 2 = 1500.

**Key points**
- A transaction groups operations: **all succeed or none do**.
- Every operation must use the same `transaction: t`.
- Flow: start → find both accounts → check balance → debit → credit → log → `commit()`.
- Any error → `rollback()` undoes everything.

---

## Q2 (d). Account statement API (4 marks)
**File:** `statement.js` (uses `bank_db`; tables must already contain data)

**Setup**
```bash
npm install express sequelize mysql2
```

**Code**
```javascript
const express = require("express");
const { Sequelize, DataTypes, Op } = require("sequelize");

const app = express();

const sequelize = new Sequelize("bank_db", "root", "your_password", {
    host: "localhost",
    dialect: "mysql"
});

const Account = sequelize.define("Account", {
    holderName: DataTypes.STRING,
    balance: DataTypes.DECIMAL(12, 2)
});

const TransferLog = sequelize.define("TransferLog", {
    fromAccountId: DataTypes.INTEGER,
    toAccountId: DataTypes.INTEGER,
    amount: DataTypes.DECIMAL(12, 2)
});

app.get("/api/accounts/:id/statement", async (req, res) => {
    const id = Number(req.params.id);

    // 1. Validate ID
    if (!Number.isInteger(id) || id <= 0) {
        return res.status(400).json({ message: "Invalid account ID" });
    }

    try {
        // 2. Check account exists
        const account = await Account.findByPk(id);
        if (!account) {
            return res.status(404).json({ message: "Account not found" });
        }

        // 3. Get sent OR received transfers, newest first
        const records = await TransferLog.findAll({
            where: {
                [Op.or]: [{ fromAccountId: id }, { toAccountId: id }]
            },
            order: [["createdAt", "DESC"]]
        });

        res.json({
            accountId: id,
            holderName: account.holderName,
            statement: records
        });
    } catch (error) {
        res.status(500).json({ message: "Unable to retrieve statement" });
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

**Test:** `GET http://localhost:3000/api/accounts/1/statement`

| Status | Meaning |
|---|---|
| 200 | Statement returned |
| 400 | Invalid account ID |
| 404 | Account does not exist |
| 500 | Database / server error |

**Key points**
- `req.params.id` reads the ID from the URL.
- `Op.or` finds transfers where the account is sender **or** receiver.
- `createdAt DESC` = newest first (Sequelize timestamps must be enabled, which is the default).

---

## Q3 (a). Custom hook useWindowSize() (5 marks)
**File:** `src/useWindowSize.js`

**Code**
```javascript
import { useEffect, useState } from "react";

function useWindowSize() {
    const [size, setSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    useEffect(() => {
        function updateSize() {
            setSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        }

        window.addEventListener("resize", updateSize);

        // Cleanup when the component is removed
        return () => {
            window.removeEventListener("resize", updateSize);
        };
    }, []);

    return size;
}

export default useWindowSize;
```

**File:** `src/App.jsx`
```jsx
import useWindowSize from "./useWindowSize";

function App() {
    const { width, height } = useWindowSize();

    return (
        <div>
            <h2>Window Size</h2>
            <p>Width: {width}px</p>
            <p>Height: {height}px</p>
        </div>
    );
}

export default App;
```

**Key points**
- `useState` holds width and height.
- `useEffect` adds the `resize` listener once.
- The `return () => {...}` cleanup removes the listener.
- Custom hook names must start with `use`.

---

## Q3 (b). Safely render course names (5 marks)
**File:** `src/CourseNames.jsx`

**Code**
```jsx
function CourseNames({ courses = [] }) {
    if (courses.length === 0) {
        return <p>No courses available</p>;
    }

    return (
        <ul>
            {courses.map((course, index) => (
                <li key={index}>{course.toUpperCase()}</li>
            ))}
        </ul>
    );
}

export default CourseNames;
```

**File:** `src/App.jsx`
```jsx
import CourseNames from "./CourseNames";

function App() {
    const courses = ["react", "node.js", "mysql"];
    return <CourseNames courses={courses} />;
}

export default App;
```

**Expected output**
- REACT
- NODE.JS
- MYSQL

If `courses` is missing or empty → `No courses available`

**Key points**
- `courses = []` is a default value if the prop is not passed.
- `length === 0` checks for an empty list.
- `map()` builds the `<li>` items; `toUpperCase()` converts text; `key` identifies each item.

---

---

# QUICK REVISION – LOGIC TO REMEMBER

## Set 2
| Q | Flow |
|---|---|
| 1(a) | GET reads → PUT validates → updates → invalid = 400 |
| 1(b) | `require("util")` → `util.inspect(object, options)` |
| 2(a) | Promise checks → `Promise.allSettled` → check each status |
| 2(b) | filter completed → calculate % → map tasks → `<progress>` |
| 2(c) | transaction → debit → credit → log → commit; error → rollback |
| 2(d) | validate ID → find account → find transfers → newest first |
| 3(a) | `useState` → `useEffect` → resize listener → update → cleanup |
| 3(b) | default `[]` → check empty → map → uppercase → key |

**END**
