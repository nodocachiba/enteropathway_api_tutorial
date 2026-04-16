# EnteroPathway API Tutorial (Local Environment)

[EnteroPathway](https://enteropathway.org) is a manually curated human gut-specific database for metabolic pathways.

This repository provides a tutorial for using the EnteroPathway REST API **in a local (self-hosted) environment**.

---

# 🚀 Goal

This README enables users to:

- Set up EnteroPathway locally
- Register a user
- Obtain a JWT token
- Successfully call API endpoints

---

# ⚠️ Important Notes (Read First)

Before starting, be aware of the following:

- JWT tokens are **environment-specific**
  - Tokens issued on `enteropathway.org` **cannot be used locally**
- API access requires BOTH:
  - ✅ Authentication (login)
  - ✅ Authorization (role assignment)
- Nginx does NOT forward Authorization headers by default
- User activation may NOT be reflected correctly in DB

---

## 🔄 Clean Start (Recommended)

Before starting, it is recommended to reset the environment:

```bash
docker compose down -v
docker compose up -d
```

## 🔧 Container Health Check

Verify all containers are running:
```bash
docker ps
docker restart <container-name>
```


# 🏗️ Step 1: User Registration

Access:

```
https://<your-server-ip>
```

Register a new account via the web UI.

---

# 🧪 Step 2: Activate User (Critical)

### Environment
Server (where Docker is running)

### Command

```bash
docker exec -it <mysql-container> mysql -u root -p
```

```sql
USE enteropathwayapp;

SELECT id, username, activated FROM user;
```

If you need the root password, you can check via this command.

```bash
docker exec -it 250918-appdb-1 env | grep MYSQL
```

---

### ❗ Problem

```
activated = 0
```

→ Cannot login

---

### ✅ Fix

```sql
UPDATE user
SET activated = 1
WHERE username = 'your_email';
```

---

# 🔐 Step 3: Assign Roles

### ❗ Problem

User exists but API returns 403

---

### Cause

No role assigned

---

### Fix

```sql
SELECT id FROM user WHERE username='your_email';

INSERT INTO user_role (user_id, roles_name)
VALUES (<user_id>, 'ROLE_USER');

INSERT INTO user_role (user_id, roles_name)
VALUES (<user_id>, 'ROLE_ADMIN');
```

---

# 🔑 Step 4: Get JWT Token

### Environment
Local machine

```bash
TOKEN=`curl -k -X POST \
-H "Content-Type: application/json" \
--data '{"username":"your_email","password":"your_password"}' \
https://<your-server-ip>/api/authenticate | \
jq '.access_token' -r`
```
The username and password in this step are the same ones which you set in Step1.

Then you can get the Token.

---

### ✅ Check

```bash
echo $TOKEN
```

---

# 🌐 Step 5: Fix Nginx (Critical)

### Environment
Server

### File

```
/DataPart/enteropathway/production_environment/src/<env>/ssl.conf
```

---

### Add

```nginx
location /api/ {
  proxy_pass http://backend:8080/api/;
  proxy_set_header Authorization $http_authorization;
}
```

---

### Restart

```bash
docker restart <nginx-container>
```

---

# ⚙️ Step 6: Fix Backend (Micronaut)

### Environment
Backend container

```bash
docker exec -it <backend-container> sh
```

### File

```
/home/app/resources/application.yml
```

---

### Add

```yaml
micronaut:
  security:
    token:
      roles-name: roles
    intercept-url-map:
      - pattern: /api/**
        access:
          - isAuthenticated()
```

---

### Restart

```bash
docker restart <backend-container>
```

---

# ✅ Step 7: API Test

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
https://<your-server-ip>/api/users
```

---

### Expected Output

```json
[
  {
    "username": "...",
    "authorities": ["ROLE_ADMIN"]
  }
]
```

---

# 🧯 Troubleshooting

## 403 Forbidden

Check:

- activated = true
- role assigned
- JWT issued locally
- nginx header forwarding
- Micronaut config

---

## 404 Not Found

Wrong endpoint:

```
/api/account ❌
/api/users ✅
```

---

## SSL Error

Use:

```bash
curl -k
```

---

# 📦 API Usage

Once API access is confirmed, refer to the original API tutorial below.

---

## Customization via JSON

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
 -H "Content-Type: application/json" \
 --data '{...}' \
 https://enteropathway.org/api/customization/mapping \
 --output mapping.pdf
```

---

## Enrichment Analysis

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
 --data '{...}' \
 https://enteropathway.org/api/customization/mapping/enrichment \
 --output enrichment.pdf
```
