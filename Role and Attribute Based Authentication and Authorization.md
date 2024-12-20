## Role-Based Authentication and Authorization
**Role-Based Authentication and Authorization (RBAC)** is a security mechanism where access to resources is granted based on roles assigned to users. Each role is associated with a set of permissions that define what actions users in that role can perform.

### Key Concepts in RBAC:
1. **Roles**: Grouping of permissions (e.g., Admin, Editor, Viewer).
2. **Permissions**: Specific actions users are allowed to perform (e.g., "edit-post", "delete-user").
3. **Users**: Individuals assigned to one or more roles.
4. **Resources**: Data or operations requiring protection (e.g., pages, APIs).
5. **Role Assignments**: Mapping users to roles.

---

### Steps to Implement RBAC in Node.js with MSSQL

#### 1. **Setup Environment**
- Install necessary packages:
  ```bash
  npm install express bcryptjs jsonwebtoken mssql dotenv
  ```
- Configure a `.env` file with your MSSQL connection details.

#### 2. **Database Design**
Create the following tables in MSSQL:

**Users Table**:
```sql
CREATE TABLE Users (
    UserId INT PRIMARY KEY IDENTITY,
    Username NVARCHAR(50) NOT NULL,
    PasswordHash NVARCHAR(255) NOT NULL,
    RoleId INT FOREIGN KEY REFERENCES Roles(RoleId)
);
```

**Roles Table**:
```sql
CREATE TABLE Roles (
    RoleId INT PRIMARY KEY IDENTITY,
    RoleName NVARCHAR(50) NOT NULL
);
```

**Permissions Table (Optional)**:
```sql
CREATE TABLE Permissions (
    PermissionId INT PRIMARY KEY IDENTITY,
    PermissionName NVARCHAR(50) NOT NULL
);

CREATE TABLE RolePermissions (
    RoleId INT FOREIGN KEY REFERENCES Roles(RoleId),
    PermissionId INT FOREIGN KEY REFERENCES Permissions(PermissionId),
    PRIMARY KEY (RoleId, PermissionId)
);
```

---

#### 3. **Backend Implementation**

**a. Connect to MSSQL:**
```javascript
const sql = require('mssql');
require('dotenv').config();

const sqlConfig = {
    user: process.env.DB_USER,
    password: process.env.DB_PASS,
    server: process.env.DB_SERVER,
    database: process.env.DB_NAME,
    options: {
        encrypt: true, // Use encryption
        trustServerCertificate: true, // For self-signed certificates
    },
};

const connectToDb = async () => {
    try {
        await sql.connect(sqlConfig);
        console.log('Connected to MSSQL');
    } catch (err) {
        console.error('Database connection failed:', err);
    }
};

connectToDb();
```

**b. User Authentication with JWT:**
```javascript
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

const SECRET_KEY = 'your-secret-key';

// Login Endpoint
app.post('/login', async (req, res) => {
    const { username, password } = req.body;

    try {
        const result = await sql.query`SELECT * FROM Users WHERE Username = ${username}`;
        const user = result.recordset[0];

        if (!user) return res.status(404).send('User not found');

        const isMatch = await bcrypt.compare(password, user.PasswordHash);
        if (!isMatch) return res.status(400).send('Invalid credentials');

        const token = jwt.sign({ userId: user.UserId, role: user.RoleId }, SECRET_KEY, {
            expiresIn: '1h',
        });
        res.json({ token });
    } catch (err) {
        res.status(500).send('Server error');
    }
});
```

**c. Middleware for Role-Based Authorization:**
```javascript
const authorize = (roles) => (req, res, next) => {
    const authHeader = req.headers['authorization'];
    if (!authHeader) return res.status(401).send('Access Denied');

    const token = authHeader.split(' ')[1];
    try {
        const decoded = jwt.verify(token, SECRET_KEY);
        req.user = decoded;

        if (!roles.includes(decoded.role)) {
            return res.status(403).send('Forbidden');
        }
        next();
    } catch (err) {
        res.status(401).send('Invalid Token');
    }
};
```

**d. Protect Routes with Role Authorization:**
```javascript
app.get('/admin', authorize(['Admin']), (req, res) => {
    res.send('Welcome Admin');
});

app.get('/editor', authorize(['Admin', 'Editor']), (req, res) => {
    res.send('Welcome Editor');
});

app.get('/viewer', authorize(['Admin', 'Editor', 'Viewer']), (req, res) => {
    res.send('Welcome Viewer');
});
```

---

#### 4. **Best Practices**
1. **Hash Passwords**: Use `bcrypt` to hash passwords before storing them in the database.
   ```javascript
   const hashPassword = async (password) => await bcrypt.hash(password, 10);
   ```
2. **Use Environment Variables**: Store sensitive information like database credentials and JWT secret in environment variables.
3. **Token Expiry**: Set a reasonable expiration for JWT tokens.
4. **Implement Permissions**: For finer control, use the `Permissions` table.
5. **Database Optimization**: Index frequently queried fields like `Username` and `RoleId`.

---

### Example `.env` File:
```plaintext
DB_USER=your_db_user
DB_PASS=your_db_password
DB_SERVER=your_db_server
DB_NAME=your_database_name
SECRET_KEY=your_secret_key
```

### Final Note:
RBAC is highly scalable and secure. Integrating it with Node.js and MSSQL provides a robust system for managing user access efficiently.

## Attribute-Based Authentication and Authorization (ABAC)
**Attribute-Based Authentication and Authorization (ABAC)** is a security mechanism where access to resources is determined by evaluating attributes associated with users, resources, actions, or the environment. ABAC provides more fine-grained control compared to Role-Based Access Control (RBAC).

---

### Key Concepts in ABAC
1. **Attributes**: Characteristics or properties associated with:
   - **Users**: E.g., age, department, job title.
   - **Resources**: E.g., document type, ownership.
   - **Actions**: E.g., read, write, delete.
   - **Environment**: E.g., time of day, location.

2. **Policy**: A set of rules that determine access. Policies evaluate attributes to decide whether access should be granted.

---

### Example: Implementing ABAC in Node.js with MSSQL

#### 1. **Database Design**

Create the necessary tables to store attributes and policies.

**Users Table**:
```sql
CREATE TABLE Users (
    UserId INT PRIMARY KEY IDENTITY,
    Username NVARCHAR(50) NOT NULL,
    Department NVARCHAR(50),
    Age INT
);
```

**Resources Table**:
```sql
CREATE TABLE Resources (
    ResourceId INT PRIMARY KEY IDENTITY,
    ResourceName NVARCHAR(50),
    OwnerId INT FOREIGN KEY REFERENCES Users(UserId)
);
```

**Policies Table**:
```sql
CREATE TABLE Policies (
    PolicyId INT PRIMARY KEY IDENTITY,
    Attribute NVARCHAR(50),
    Condition NVARCHAR(50),
    Value NVARCHAR(50),
    Action NVARCHAR(50)
);
```

---

#### 2. **Backend Implementation**

**a. Setup and MSSQL Connection**
```javascript
const sql = require('mssql');
require('dotenv').config();

const sqlConfig = {
    user: process.env.DB_USER,
    password: process.env.DB_PASS,
    server: process.env.DB_SERVER,
    database: process.env.DB_NAME,
    options: {
        encrypt: true,
        trustServerCertificate: true,
    },
};

const connectToDb = async () => {
    try {
        await sql.connect(sqlConfig);
        console.log('Connected to MSSQL');
    } catch (err) {
        console.error('Database connection failed:', err);
    }
};

connectToDb();
```

**b. Function to Evaluate Policies**
```javascript
const evaluatePolicy = async (userId, resourceId, action) => {
    try {
        // Fetch user and resource attributes
        const userResult = await sql.query`SELECT * FROM Users WHERE UserId = ${userId}`;
        const resourceResult = await sql.query`SELECT * FROM Resources WHERE ResourceId = ${resourceId}`;
        
        const user = userResult.recordset[0];
        const resource = resourceResult.recordset[0];

        if (!user || !resource) return false;

        // Fetch policies
        const policies = await sql.query`SELECT * FROM Policies WHERE Action = ${action}`;

        // Evaluate each policy
        for (const policy of policies.recordset) {
            const { Attribute, Condition, Value } = policy;

            // Check conditions based on attributes
            const userAttribute = user[Attribute] || resource[Attribute];
            if (!userAttribute) continue;

            switch (Condition) {
                case '=':
                    if (userAttribute == Value) return true;
                    break;
                case '>':
                    if (userAttribute > Value) return true;
                    break;
                case '<':
                    if (userAttribute < Value) return true;
                    break;
                // Add more conditions as needed
                default:
                    continue;
            }
        }

        return false; // Deny access if no policy matches
    } catch (err) {
        console.error('Policy evaluation failed:', err);
        return false;
    }
};
```

**c. Middleware for Attribute-Based Authorization**
```javascript
const authorize = (action) => async (req, res, next) => {
    const { userId, resourceId } = req.body; // Assume these are passed in the request

    const isAuthorized = await evaluatePolicy(userId, resourceId, action);

    if (!isAuthorized) {
        return res.status(403).send('Access Denied');
    }
    next();
};
```

**d. Protect Routes Using ABAC**
```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.post('/view-resource', authorize('read'), (req, res) => {
    res.send('Resource accessed');
});

app.post('/edit-resource', authorize('write'), (req, res) => {
    res.send('Resource edited');
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

---

#### 3. **Example Data**

**Insert Users:**
```sql
INSERT INTO Users (Username, Department, Age) VALUES
('Alice', 'HR', 30),
('Bob', 'Engineering', 25);
```

**Insert Resources:**
```sql
INSERT INTO Resources (ResourceName, OwnerId) VALUES
('Document1', 1),
('Document2', 2);
```

**Insert Policies:**
```sql
INSERT INTO Policies (Attribute, Condition, Value, Action) VALUES
('Department', '=', 'HR', 'read'),
('Age', '>', '25', 'write');
```

---

#### 4. **Testing the Implementation**

**Scenario 1**: Alice (HR department) tries to read a document.
- Policy matches: `Department = 'HR'` and action `read`.
- Access granted.

**Scenario 2**: Bob (Engineering, Age 25) tries to write to a document.
- Policy requires `Age > 25` for action `write`.
- Access denied.

---

### Summary

ABAC provides fine-grained control by evaluating multiple attributes and policies. The flexibility of ABAC makes it suitable for complex authorization scenarios. The example demonstrates how to integrate ABAC in a Node.js application with MSSQL.
