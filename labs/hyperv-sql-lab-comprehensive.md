# Hyper-V Lab — Active Directory × SQL Server × SSMS

## Lab Environment

```
                   Active Directory
                      (AD Server)
                         │
                         │ Domain / Authentication
                         │
        ┌────────────────┴────────────────┐
        │                                 │
   User PC 1                         User PC 2
   (SSMS)                            (SSMS)
        │                                 │
        └──────────────┬──────────────────┘
                       │
                  TCP 1433
                       │
                       ▼
                 SQL Server
                    (SQL01)
```

### Main components

**AD Server**
- Domain: DEV
- Manages users, groups, and computer accounts

**SQL Server**
- Server name: SQL01
- SQL Server listens on TCP 1433

**User PC**
- Domain-joined client
- SSMS installed
- Used to remotely manage SQL Server

---

## 2. Active Directory Management

Active Directory Users and Computers (ADUC) is used to manage:
- Users
- Groups
- Computer accounts
- Organizational Units (OUs)
- User/group membership
- Password resets
- Account enable/disable

**Example:**
```
testdev2
   │
   ├── SQLDevelopers
   │
   └── testgroup
```

> Adding a user to an AD group means that the user becomes a member of that group.  
> However: AD group membership alone does NOT automatically give the group access to SQL Server.

---

## 3. Checking Group Membership

### Method 1: Check groups in current Windows logon session
```cmd
whoami /groups
```

**Example output:**
```
DEV\SQLDevelopers
DEV\testgroup
```
This confirms that the currently logged-in user has both groups in their session.

### Method 2: Check AD membership directly (from DC01 / PowerShell)
```powershell
Get-ADUser testdev2 -Properties MemberOf |
Select-Object -ExpandProperty MemberOf
```

### Important distinction:
- `Get-ADUser` → AD上で設定されているグループ所属
- `whoami /groups` → 現在のWindowsログオンセッションが持っているグループ情報

> If a user was recently added to a group, signing out and signing back in may be necessary for the new group membership to appear in the user's security token.

---

## 4. SQL Server Network Connectivity

SQL Server can communicate over TCP/IP. For the default SQL Server configuration, TCP port 1433 is commonly used.

**Network flow:**
```
User PC
   │
   │ TCP 1433
   ▼
SQL01
   │
   ▼
SQL Server
TCP 1433
```

**TCP 1433 is:**
- The network endpoint/port through which SQL Server accepts TCP connections.
- NOT a user permission, AD permission, database permission, or authentication method

**Security requires:**
- Firewall rules
- Network restrictions
- Authentication
- Authorization
- Encryption/TLS

---

## 5. Testing TCP Connectivity

From the User PC:
```cmd
Test-NetConnection SQL01 -Port 1433
```

**If the result contains:**
```
TcpTestSucceeded : True
```

Then:
- The User PC can reach SQL01 on TCP port 1433
- This only proves network connectivity
- It does NOT prove: user authentication, SQL Server Login, database access, or specific operation permissions

---

## 6. Connecting with SSMS

From the User PC, SQL Server Management Studio (SSMS) can be used to remotely manage SQL Server.

**Server name example:**
```
SQL01,1433
```

This means:
```
SQL01 → Connect to this server
1433  → Use TCP port 1433
```

**Connection process:**
```
User PC
   │
   │ Resolve SQL01
   │
   │ TCP 1433
   ▼
SQL Server
   │
   │ Authentication
   ▼
Windows / AD identity
   │
   ▼
SQL Server Login
   │
   ▼
Database User Mapping
   │
   ▼
Database Role
   │
   ▼
Database operation
```

SSMS is a client/admin tool that:
- Connects to SQL Server
- Sends commands/queries
- Receives results
- Manages SQL Server configuration

---

## 7. AD Group vs SQL Server Login

**This is one of the most important concepts in this lab.**

Creating an AD group (`DEV\testgroup`) does NOT automatically create a SQL Server Login.

**The relationship is:**
```
Active Directory
────────────────────────

DEV\testgroup
      │
      └── testdev2


SQL Server
────────────────────────

Security
  └── Logins
        └── DEV\testgroup
```

**The AD group must be registered as a SQL Server Login if it needs SQL Server access.**

In SSMS:
```
SQL Server
  └── Security
       └── Logins
            └── New Login
```

Then specify: `DEV\testgroup`

---

## 8. SQL Server Login → User Mapping

Creating a SQL Server Login does not necessarily mean the user can access every database. The next layer is User Mapping.

**Conceptual flow:**
```
AD Group
   ↓
SQL Server Login
   ↓
Database User
   ↓
Database Role
```

**Example:**
```
DEV\testgroup
      ↓
SQL Server Login
      ↓
Database: MyDatabase
      ↓
db_datareader
db_datawriter
```

This determines what the group can actually do inside the database.

---

## 9. Database Roles

Database roles determine permissions within a database.

**Common examples:**
- `db_datareader` → Can read data
- `db_datawriter` → Can insert/update/delete data

**Therefore:**
- Authentication and authorization are separate concepts
- Authentication: "Who are you?"
- Authorization: "What are you allowed to do?"

---

## 10. TCP Port vs User Permission

**A key lesson from this lab:**

Adding a new user to an existing AD group does NOT require a new TCP port.

```
TCP 1433
    ↓
SQL Server network access
is independent from

testdev2
    ↓
SQLDevelopers
    ↓
SQL Server Login
    ↓
Database permissions
```

- TCP 1433 controls the network path
- AD/SQL permissions control identity and authorization

---

## 11. Troubleshooting Example

**Problem:** A user was added to an AD group, but the expected group information was not immediately visible.

**Investigation steps:**

1. Check domain connectivity
   ```cmd
   Test-NetConnection <AD-IP>
   ```

2. Check SQL connectivity
   ```cmd
   Test-NetConnection SQL01 -Port 1433
   ```

3. Check domain controller discovery
   ```cmd
   nltest /dsgetdc:DEV
   ```

4. Check current Windows group membership
   ```cmd
   whoami /groups
   ```

5. Check AD group membership
   ```powershell
   Get-ADUser testdev2 -Properties MemberOf
   ```

6. Sign out / sign in again
   - This refreshes the Windows logon session and its security token

**After logging in again:**
```cmd
whoami /groups
```
confirmed the expected group membership.

---

## 12. Important Mental Model

The entire system can be understood as **five layers**:

### Layer 1: Network
```
Can I reach SQL Server?

   TCP 1433
   Firewall
```

### Layer 2: Identity
```
Who am I?

   Active Directory
   Windows Authentication
```

### Layer 3: SQL Server Login
```
Does SQL Server recognize this user/group?

   Security → Logins
```

### Layer 4: Database Access
```
Which database can I access?

   User Mapping
```

### Layer 5: Database Permissions
```
What can I do?

   Database Roles
```

> A successful connection requires these layers to work together.

---

## 13. Key Lessons

### Lesson 1
AD manages identities, groups, and computers.

### Lesson 2
AD group membership does not automatically create a SQL Server Login.

### Lesson 3
TCP 1433 provides the network path to SQL Server; it does not provide authorization.

### Lesson 4
`Test-NetConnection SQL01 -Port 1433` verifies network connectivity, not database permissions.

### Lesson 5
`whoami /groups` shows the groups available in the current Windows logon session.

### Lesson 6
SQL Server access can be managed through an AD group instead of creating individual SQL permissions for every user.

**Example:**
```
AD

SQLDevelopers
├── testdev
├── testdev2
└── testdev3


SQL Server

Security
└── Logins
    └── DEV\SQLDevelopers
```

All members can inherit the SQL Server permissions assigned to that group.

### Lesson 7
The overall authentication/authorization chain is:

```
AD User
   ↓
AD Group
   ↓
SQL Server Login
   ↓
Database User Mapping
   ↓
Database Role
   ↓
Actual Database Permissions
```

This separation between network connectivity, authentication, and authorization is a fundamental infrastructure concept and also applies to cloud environments such as Azure.

---

## 📋 Quick Reference Commands

```cmd
REM Check AD group membership in current session
whoami /groups

REM Check if TCP port 1433 is reachable
Test-NetConnection SQL01 -Port 1433

REM Refresh group membership (sign out required for full effect)
logoff

REM Check AD user's direct group membership
Get-ADUser testdev2 -Properties MemberOf | Select-Object -ExpandProperty MemberOf
```

---

*Document created for Hyper-V Hands-On Lab documentation*
*Focus: Integrating Active Directory, SQL Server, and Windows Authentication*