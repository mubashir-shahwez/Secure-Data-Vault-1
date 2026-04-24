# Secure Data Vault

A desktop application for securely storing sensitive text secrets and encrypted files using AES-256-CBC encryption, PBKDF2 password hashing, and MySQL persistence.

---

## Table of Contents

- [Features](#features)
- [Technology Stack](#technology-stack)
- [Security Architecture](#security-architecture)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Database Setup](#database-setup)
- [Configuration](#configuration)
- [Running the Application](#running-the-application)
- [Usage Guide](#usage-guide)
- [Project Structure](#project-structure)
- [Security Notes](#security-notes)
- [Known Limitations](#known-limitations)
- [Future Enhancements](#future-enhancements)

---

## Features

### Core Functionality
- **User Authentication** — Register and login with PBKDF2-HMAC-SHA256 hashed passwords (100,000 iterations)
- **Text Secret Management** — Create, read, update, and delete encrypted text secrets with AES-256-CBC
- **File Encryption** — Upload and encrypt files up to 10 MB with in-app preview for images and text
- **Search & Filter** — Search secrets by key name with real-time filtering
- **Audit Trail** — Comprehensive access logging for all operations (VIEW, ADD, UPDATE, DELETE, DOWNLOAD_FILE)

### Security Features
- AES-256-CBC encryption with a unique random IV per operation
- PBKDF2-HMAC-SHA256 key derivation with 100,000 iterations
- Salted password hashing stored in `salt:hash` format
- Decrypt-on-demand — plaintext only exists in memory during view operations
- Ciphertext verification via "View Cipher" and "File Cipher" buttons

### User Experience
- Clean Java Swing GUI with table views and dialogs
- In-app file preview for images (PNG, JPG, GIF) and text files (TXT, CSV, LOG)
- Multi-user support with data isolation enforced via foreign key constraints

---

## Technology Stack

| Component | Technology |
|---|---|
| Language | Java 17+ |
| GUI Framework | Java Swing |
| Database | MySQL 8.0+ |
| JDBC Driver | MySQL Connector/J 8.0+ |
| Encryption | Java Cryptography Extension (JCE) |
| Build Tool | IntelliJ IDEA / Manual compilation |

---

## Security Architecture

### Password Storage
- **Algorithm** — PBKDF2-HMAC-SHA256
- **Iterations** — 100,000
- **Salt** — 16-byte random salt per user
- **Storage Format** — `salt:hash` (both Base64-encoded) stored in `VARCHAR(512)`

### Data Encryption
- **Algorithm** — AES-256-CBC with PKCS5 padding
- **Key Derivation** — PBKDF2-HMAC-SHA256 derived from the master password
- **IV** — 16-byte random IV generated fresh per encryption operation
- **Text Storage Format** — `Base64(IV || CIPHERTEXT)` stored as TEXT in database
- **File Storage Format** — Raw bytes `IV || CIPHERTEXT` stored as LONGBLOB

### Data Flow

```
Login
  └── Master Password → PBKDF2 → 256-bit AES Key

Storing a Secret
  └── Plaintext → AES-256-CBC (random IV) → Base64(IV || Ciphertext) → MySQL

Retrieving a Secret
  └── Base64 Ciphertext → Split IV + Ciphertext → AES Decrypt → Plaintext (in memory only)

File Upload
  └── Raw Bytes → AES-256-CBC (random IV) → IV || Ciphertext → LONGBLOB in MySQL

File Download
  └── LONGBLOB → Split IV + Ciphertext → AES Decrypt → Original File Bytes
```

---

## Prerequisites

Before running the project, make sure you have the following installed:

### 1. Java Development Kit (JDK) 17 or higher

**Check if already installed:**
```bash
java -version
javac -version
```

**Install JDK 17:**
- **Windows / macOS / Linux** — Download from [https://adoptium.net](https://adoptium.net) (Temurin JDK 17, recommended)
- Or via package manager:
  ```bash
  # Ubuntu/Debian
  sudo apt install openjdk-17-jdk

  # macOS (Homebrew)
  brew install openjdk@17

  # Windows
  # Download and run the installer from https://adoptium.net
  ```

### 2. MySQL Server 8.0 or higher

**Check if already installed:**
```bash
mysql --version
```

**Install MySQL:**
- **Windows** — Download MySQL Installer from [https://dev.mysql.com/downloads/installer](https://dev.mysql.com/downloads/installer)
- **macOS** — `brew install mysql`
- **Ubuntu/Debian** — `sudo apt install mysql-server`

After installation, start the MySQL service:
```bash
# Linux
sudo systemctl start mysql

# macOS
brew services start mysql

# Windows
# MySQL service starts automatically after installation
# Or open Services and start "MySQL80"
```

### 3. MySQL Connector/J (JDBC Driver)

Download the MySQL Connector/J 8.0+ JAR from:
[https://dev.mysql.com/downloads/connector/j](https://dev.mysql.com/downloads/connector/j)

Select **Platform Independent** and download the ZIP. Extract it and keep the `.jar` file — you will need it in the classpath.

---

## Installation & Setup

### Step 1 — Clone the Repository

```bash
git clone https://github.com/mubashirshahwez/secure-data-vault.git
cd secure-data-vault
```

### Step 2 — Place the MySQL Connector JAR

Copy the MySQL Connector/J JAR file into the project root or a `lib/` folder:

```
secure-data-vault/
├── lib/
│   └── mysql-connector-j-8.x.x.jar   <-- place it here
├── DataSecureVault/
│   └── src/
└── ...
```

---

## Database Setup

### Step 1 — Log into MySQL

```bash
mysql -u root -p
```
Enter your MySQL root password when prompted.

### Step 2 — Create the Database

```sql
CREATE DATABASE vaultdb;
USE vaultdb;
```

### Step 3 — Run the Schema

Exit MySQL and run the schema file from the project directory:

```bash
mysql -u root -p vaultdb < DataSecureVault/schema.sql
```

Or paste the contents of `schema.sql` directly inside the MySQL shell. This will create three tables:

| Table | Purpose |
|---|---|
| `users` | Stores usernames and hashed passwords |
| `vault_data` | Stores encrypted text secrets |
| `access_logs` | Stores audit logs for all operations |

**Note:** The schema also creates a `vault_files` table used by VaultFileService for encrypted file storage. If the table is not present in `schema.sql`, run the following manually:

```sql
CREATE TABLE vault_files (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50),
    file_size INT NOT NULL,
    encrypted_data LONGBLOB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Step 4 — Verify Tables

```sql
USE vaultdb;
SHOW TABLES;
```

Expected output:
```
+-------------------+
| Tables_in_vaultdb |
+-------------------+
| access_logs       |
| users             |
| vault_data        |
| vault_files       |
+-------------------+
```

---

## Configuration

Open `DataSecureVault/config.properties` and update the database credentials to match your local MySQL setup:

```properties
# Database Configuration
db.type=mysql
db.url=jdbc:mysql://localhost:3306/vaultdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
db.username=root
db.password=your_mysql_password_here

# Security Settings
encryption.iterations=65536
encryption.key.length=256
```

Replace `your_mysql_password_here` with your actual MySQL root password.

**Important:** If your MySQL username is not `root`, update `db.username` accordingly.

---

## Running the Application

### Option 1 — Using IntelliJ IDEA (Recommended)

1. Open IntelliJ IDEA
2. Click **File → Open** and select the `DataSecureVault/` folder
3. Go to **File → Project Structure → Libraries**
4. Click `+` → **Java** and add the MySQL Connector/J JAR
5. Set the Project SDK to JDK 17 under **File → Project Structure → Project**
6. Open `src/com/vault/Main.java`
7. Right-click → **Run 'Main'**

### Option 2 — Manual Compilation and Run (Command Line)

**Step 1 — Compile all source files:**

```bash
cd DataSecureVault

javac -cp ".:lib/mysql-connector-j-8.x.x.jar" \
  -d out \
  src/com/vault/Main.java \
  src/com/vault/core/*.java \
  src/com/vault/model/*.java \
  src/com/vault/service/*.java \
  src/com/vault/ui/*.java
```

On **Windows**, replace `:` with `;` in the classpath:

```bash
javac -cp ".;lib\mysql-connector-j-8.x.x.jar" ^
  -d out ^
  src\com\vault\Main.java ^
  src\com\vault\core\*.java ^
  src\com\vault\model\*.java ^
  src\com\vault\service\*.java ^
  src\com\vault\ui\*.java
```

**Step 2 — Copy config.properties to the output directory:**

```bash
cp config.properties out/
```

**Step 3 — Run the application:**

```bash
# Linux / macOS
java -cp "out:lib/mysql-connector-j-8.x.x.jar" com.vault.Main

# Windows
java -cp "out;lib\mysql-connector-j-8.x.x.jar" com.vault.Main
```

### Option 3 — Using Eclipse

1. Open Eclipse → **File → Import → Existing Projects into Workspace**
2. Select the `DataSecureVault/` folder
3. Right-click the project → **Build Path → Add External Archives**
4. Add the MySQL Connector/J JAR
5. Right-click `Main.java` → **Run As → Java Application**

---

## Usage Guide

### Registration
1. Launch the application
2. Enter a username and a password (minimum 6 characters)
3. Click **Register**
4. Your password is hashed immediately — it is never stored in plaintext

### Login
1. Enter your registered username and master password
2. Click **Login**
3. The same password is used to derive your AES encryption key — keep it safe

### Managing Secrets
| Action | How |
|---|---|
| Add a secret | Click **Add Secret**, enter a key name and value, click Save |
| View a secret | Select a row, click **View Secret** — decrypted value shown in memory only |
| Update a secret | Select a row, click **Update**, modify and save |
| Delete a secret | Select a row, click **Delete**, confirm |
| Search secrets | Type in the search bar and press the search button |
| View raw ciphertext | Select a row, click **View Cipher** |

### Managing Files
| Action | How |
|---|---|
| Upload a file | Click **Add File**, select a file (max 10 MB) |
| Download a file | Click **View Files**, select a file, choose save location |
| Preview a file | Click **Preview File**, select an image or text file |
| View file ciphertext | Click **File Cipher**, select a file |

---

## Project Structure

```
secure-data-vault/
└── DataSecureVault/
    ├── config.properties          # Database and security configuration
    ├── schema.sql                 # MySQL table definitions
    ├── DataSecureVault.iml        # IntelliJ module file
    └── src/
        └── com/
            └── vault/
                ├── Main.java                     # Entry point
                ├── core/
                │   ├── DatabaseManager.java      # Singleton DB connection manager
                │   └── EncryptionManager.java    # AES-256 + PBKDF2 implementation
                ├── model/
                │   ├── User.java                 # User entity
                │   ├── Secret.java               # Secret entity
                │   └── AccessLog.java            # Audit log entity
                ├── service/
                │   ├── UserService.java           # Registration, login, logging
                │   ├── VaultService.java          # Secret CRUD operations
                │   └── VaultFileService.java      # File encryption and retrieval
                └── ui/
                    ├── LoginFrame.java            # Login and registration screen
                    ├── MainVaultFrame.java        # Main dashboard
                    ├── AddSecretDialog.java       # Add secret dialog
                    ├── UpdateSecretDialog.java    # Update secret dialog
                    ├── ViewSecretDialog.java      # View decrypted secret
                    ├── ViewCipherDialog.java      # View raw ciphertext (text)
                    └── ViewFileCipherDialog.java  # View raw ciphertext (file)
```

---

## Security Notes

- **Master password is never stored** — it is only used to derive the AES key via PBKDF2 at runtime
- **Each encryption operation uses a fresh random IV** — identical plaintexts produce different ciphertexts
- **Plaintext only exists in memory** during view operations and is never written to disk or database
- **Constant-time comparison** is used during password verification to prevent timing attacks
- **Foreign key constraints** enforce strict user-level data isolation in the database
- `config.properties` contains your database password — do not commit this file to a public repository. Add it to `.gitignore`:
  ```
  config.properties
  ```

---

## Known Limitations

- The encryption salt used for key derivation in `EncryptionManager` is currently a static string (`"a9v5n38s"`). For production use, this should be a per-user random salt stored securely alongside the encrypted data.
- File size is capped at 10 MB. Larger files require BLOB storage tuning in MySQL (`max_allowed_packet`).
- The application uses a single persistent database connection. High concurrency is not supported in the current design.
- No session timeout is implemented — the vault remains open until the user explicitly logs out.

---

## Future Enhancements

- Per-user random salt for key derivation
- Session timeout with automatic re-authentication
- Export/import vault backup (encrypted)
- Two-factor authentication support
- Cross-platform packaging (JAR, EXE, DMG)
- Migration to a connection pool (HikariCP) for better concurrency

---


License
This project is open source and available under the MIT License.

## License

This project is open source and available under the [MIT License](LICENSE).
