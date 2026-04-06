# 📘 Aliado Legal API — Documentação Completa

> **Base URL (desenvolvimento):** `http://localhost:8000`  
> **Base URL (produção):** `https://aliadolegal-app-3uhfe.ondigitalocean.app`  
> **Framework:** FastAPI (Python)  
> **Autenticação:** OAuth2 com Bearer Token (JWT)

---

## 📑 Índice

1. [Visão Geral](#visão-geral)
2. [Autenticação](#autenticação)
3. [Configuração do Cliente HTTP](#configuração-do-cliente-http)
4. [Endpoints](#endpoints)
   - [Root](#1-root)
   - [Auth](#2-auth)
   - [Pluggy](#3-pluggy)
   - [Webhooks](#4-webhooks)
   - [Queries](#5-queries)
5. [Schemas](#schemas)
6. [Tratamento de Erros](#tratamento-de-erros)

---

## Visão Geral

A **Aliado Legal API** é uma API RESTful construída com FastAPI que gerencia:

- **Autenticação de usuários** com criptografia AES e JWT
- **Integração com Pluggy** para dados financeiros (contas, transações, investimentos, empréstimos)
- **Webhooks** para receber eventos do Pluggy
- **Consultas consolidadas** com auditoria de acesso

### Fluxo de Autenticação

```mermaid
sequenceDiagram
    participant Frontend
    participant API
    participant DB

    Frontend->>API: POST /auth/login (email + senha criptografada)
    API->>API: Descriptografa senha (AES)
    API->>DB: Busca usuário por email
    API->>API: Verifica senha (bcrypt)
    API-->>Frontend: { access_token, token_type: "bearer" }
    Frontend->>API: GET /auth/me (Authorization: Bearer <token>)
    API-->>Frontend: { id, email, cpf, created_at }
```

---

## Autenticação

Todos os endpoints (exceto `GET /`, `GET /health`, `POST /auth/register`, `POST /auth/login` e `POST /webhooks/pluggy`) exigem autenticação via **Bearer Token**.

### Header de Autenticação

```
Authorization: Bearer <access_token>
```

O token é obtido via `POST /auth/login` e tem validade padrão de **30 minutos**.

> [!IMPORTANT]
> As senhas enviadas ao backend devem ser **criptografadas com AES** usando o `INVITE_TOKEN` como chave, via CryptoJS no frontend.

---

## Configuração do Cliente HTTP

### React / TypeScript — API Client

```typescript
// src/services/api.ts

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000";

interface RequestOptions {
  method?: string;
  headers?: Record<string, string>;
  body?: any;
}

class ApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  private getToken(): string | null {
    if (typeof window !== "undefined") {
      return localStorage.getItem("access_token");
    }
    return null;
  }

  private async request<T>(endpoint: string, options: RequestOptions = {}): Promise<T> {
    const token = this.getToken();
    const headers: Record<string, string> = {
      "Content-Type": "application/json",
      ...options.headers,
    };

    if (token) {
      headers["Authorization"] = `Bearer ${token}`;
    }

    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: options.method || "GET",
      headers,
      body: options.body ? JSON.stringify(options.body) : undefined,
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({ detail: "Erro desconhecido" }));
      throw new Error(error.detail || `HTTP ${response.status}`);
    }

    return response.json();
  }

  // Métodos auxiliares
  get<T>(endpoint: string) {
    return this.request<T>(endpoint, { method: "GET" });
  }

  post<T>(endpoint: string, body?: any) {
    return this.request<T>(endpoint, { method: "POST", body });
  }

  // Login usa form-urlencoded (padrão OAuth2)
  async login(email: string, encryptedPassword: string) {
    const formData = new URLSearchParams();
    formData.append("username", email);
    formData.append("password", encryptedPassword);

    const response = await fetch(`${this.baseUrl}/auth/login`, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: formData.toString(),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail);
    }

    const data = await response.json();
    localStorage.setItem("access_token", data.access_token);
    return data;
  }
}

export const api = new ApiClient(API_BASE_URL);
```

### Node.js — API Client

```javascript
// api-client.js

const API_BASE_URL = process.env.API_BASE_URL || "http://localhost:8000";

class ApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.accessToken = null;
  }

  async request(endpoint, options = {}) {
    const headers = {
      "Content-Type": "application/json",
      ...options.headers,
    };

    if (this.accessToken) {
      headers["Authorization"] = `Bearer ${this.accessToken}`;
    }

    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: options.method || "GET",
      headers,
      body: options.body ? JSON.stringify(options.body) : undefined,
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({ detail: "Unknown error" }));
      throw new Error(error.detail || `HTTP ${response.status}`);
    }

    return response.json();
  }

  async login(email, encryptedPassword) {
    const formData = new URLSearchParams();
    formData.append("username", email);
    formData.append("password", encryptedPassword);

    const response = await fetch(`${this.baseUrl}/auth/login`, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: formData.toString(),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail);
    }

    const data = await response.json();
    this.accessToken = data.access_token;
    return data;
  }
}

const api = new ApiClient(API_BASE_URL);
module.exports = { api };
```

---

## Endpoints

---

### 1. Root

#### `GET /` — Health Check Simples

Verifica se a API está online.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /` |
| **Auth** | ❌ Não |
| **Content-Type** | — |

**Response `200 OK`:**
```json
{
  "message": "Aliado Legal API online"
}
```

**Insomnia:**
```
GET http://localhost:8000/
```
> Sem headers adicionais. Basta enviar a requisição.

**React / TypeScript:**
```typescript
const status = await api.get<{ message: string }>("/");
console.log(status.message); // "Aliado Legal API online"
```

**Node.js:**
```javascript
const status = await api.request("/");
console.log(status.message); // "Aliado Legal API online"
```

---

#### `GET /health` — Health Check Detalhado

| Campo | Valor |
|-------|-------|
| **URL** | `GET /health` |
| **Auth** | ❌ Não |

**Response `200 OK`:**
```json
{
  "status": "ok"
}
```

**Insomnia:**
```
GET http://localhost:8000/health
```

**React / TypeScript:**
```typescript
const health = await api.get<{ status: string }>("/health");
```

**Node.js:**
```javascript
const health = await api.request("/health");
```

---

### 2. Auth

#### `POST /auth/register` — Registrar Novo Usuário

Cria um novo usuário no sistema. Requer um **token de convite** válido.

| Campo | Valor |
|-------|-------|
| **URL** | `POST /auth/register` |
| **Auth** | ❌ Não |
| **Content-Type** | `application/json` |

**Request Body:**
```json
{
  "email": "usuario@email.com",
  "cpf": "12345678909",
  "password": "U2FsdGVkX1+...(senha criptografada AES)...",
  "invite_token": "meu_token_secreto_aws"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `email` | `string (email)` | ✅ | Email válido do usuário |
| `cpf` | `string` | ✅ | CPF (11 dígitos) ou CNPJ (14 dígitos), validado matematicamente |
| `password` | `string` | ✅ | Senha criptografada com AES (CryptoJS) |
| `invite_token` | `string` | ✅ | Token de convite para autorizar o cadastro |

**Response `200 OK`:**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "usuario@email.com",
  "cpf": "12345678909",
  "created_at": "2026-04-02T19:00:00Z",
  "updated_at": null
}
```

**Erros Possíveis:**

| Status | Detalhe |
|--------|---------|
| `403` | Token de convite inválido |
| `400` | Email já existe no sistema |
| `400` | CPF/CNPJ já existe no sistema |
| `400` | Falha ao descriptografar a senha |
| `422` | CPF/CNPJ inválido (validação matemática) |

**Insomnia:**
```
POST http://localhost:8000/auth/register

Headers:
  Content-Type: application/json

Body (JSON):
{
  "email": "novo@email.com",
  "cpf": "12345678909",
  "password": "U2FsdGVkX1+abc123encryptedpassword==",
  "invite_token": "meu_token_secreto_aws"
}
```

**React / TypeScript:**
```typescript
import CryptoJS from "crypto-js";

const INVITE_TOKEN = "meu_token_secreto_aws";

async function registerUser(email: string, cpf: string, password: string) {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const response = await api.post<{
    id: string;
    email: string;
    cpf: string;
    created_at: string;
    updated_at: string | null;
  }>("/auth/register", {
    email,
    cpf,
    password: encryptedPassword,
    invite_token: INVITE_TOKEN,
  });

  return response;
}

// Uso:
const user = await registerUser("novo@email.com", "12345678909", "minhaSenha123");
console.log(user.id);
```

**Node.js:**
```javascript
const CryptoJS = require("crypto-js");

const INVITE_TOKEN = "meu_token_secreto_aws";

async function registerUser(email, cpf, password) {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const response = await fetch("http://localhost:8000/auth/register", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      email,
      cpf,
      password: encryptedPassword,
      invite_token: INVITE_TOKEN,
    }),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(err.detail);
  }

  return response.json();
}
```

---

#### `POST /auth/login` — Fazer Login

Autentica o usuário e retorna um JWT token.

| Campo | Valor |
|-------|-------|
| **URL** | `POST /auth/login` |
| **Auth** | ❌ Não |
| **Content-Type** | `application/x-www-form-urlencoded` |

> [!WARNING]
> Este endpoint usa **`application/x-www-form-urlencoded`**, NÃO `application/json`. É o padrão OAuth2 do FastAPI.

**Request Body (form-urlencoded):**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `username` | `string` | ✅ | Email do usuário |
| `password` | `string` | ✅ | Senha criptografada com AES |

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

**Erros Possíveis:**

| Status | Detalhe |
|--------|---------|
| `400` | Incorrect email or password |

**Insomnia:**
```
POST http://localhost:8000/auth/login

Headers:
  Content-Type: application/x-www-form-urlencoded

Body (Form URL Encoded):
  username = usuario@email.com
  password = U2FsdGVkX1+abc123encryptedpassword==
```

> **Configuração no Insomnia:**
> 1. Selecione o método **POST**
> 2. No Body, escolha **Form URL Encoded**
> 3. Adicione os campos `username` e `password`

**React / TypeScript:**
```typescript
import CryptoJS from "crypto-js";

const INVITE_TOKEN = "meu_token_secreto_aws";

async function login(email: string, password: string): Promise<string> {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const formData = new URLSearchParams();
  formData.append("username", email);
  formData.append("password", encryptedPassword);

  const response = await fetch("http://localhost:8000/auth/login", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: formData.toString(),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.detail);
  }

  const data: { access_token: string; token_type: string } = await response.json();
  localStorage.setItem("access_token", data.access_token);
  return data.access_token;
}
```

**Node.js:**
```javascript
const CryptoJS = require("crypto-js");

const INVITE_TOKEN = "meu_token_secreto_aws";

async function login(email, password) {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const formData = new URLSearchParams();
  formData.append("username", email);
  formData.append("password", encryptedPassword);

  const response = await fetch("http://localhost:8000/auth/login", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: formData.toString(),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(err.detail);
  }

  const data = await response.json();
  api.accessToken = data.access_token; // salva no client
  return data;
}
```

---

#### `GET /auth/me` — Obter Usuário Logado

Retorna os dados do usuário autenticado.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /auth/me` |
| **Auth** | ✅ Bearer Token |

**Response `200 OK`:**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "usuario@email.com",
  "cpf": "12345678909",
  "created_at": "2026-04-02T19:00:00Z",
  "updated_at": null
}
```

**Erros Possíveis:**

| Status | Detalhe |
|--------|---------|
| `401` | Could not validate credentials |

**Insomnia:**
```
GET http://localhost:8000/auth/me

Headers:
  Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

> **Configuração no Insomnia:**
> 1. Na aba **Auth**, selecione **Bearer Token**
> 2. Cole o token obtido no login

**React / TypeScript:**
```typescript
interface UserResponse {
  id: string;
  email: string;
  cpf: string;
  created_at: string;
  updated_at: string | null;
}

const currentUser = await api.get<UserResponse>("/auth/me");
console.log(currentUser.email);
```

**Node.js:**
```javascript
const currentUser = await api.request("/auth/me");
console.log(currentUser.email);
```

---

### 3. Pluggy

> [!NOTE]
> Todos os endpoints Pluggy requerem **autenticação Bearer Token**. Eles sincronizam dados com a API externa do Pluggy e persistem localmente no banco de dados.

---

#### `POST /pluggy/connect-token` — Gerar Token de Conexão

Gera um token para ser usado no widget Pluggy Connect no frontend.

| Campo | Valor |
|-------|-------|
| **URL** | `POST /pluggy/connect-token` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `item_id` (opcional) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `item_id` | `string` | ❌ | ID de um item existente para reconexão |

**Response `200 OK`:**
```json
{
  "accessToken": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9..."
}
```

**Insomnia:**
```
POST http://localhost:8000/pluggy/connect-token

Headers:
  Authorization: Bearer <seu_token>

-- OU para reconexão: --
POST http://localhost:8000/pluggy/connect-token?item_id=abc123
```

**React / TypeScript:**
```typescript
interface ConnectTokenResponse {
  accessToken: string;
}

// Nova conexão
const token = await api.post<ConnectTokenResponse>("/pluggy/connect-token");
console.log(token.accessToken);

// Reconexão de item existente
const reconnectToken = await api.post<ConnectTokenResponse>(
  "/pluggy/connect-token?item_id=abc123"
);
```

**Node.js:**
```javascript
// Nova conexão
const token = await api.request("/pluggy/connect-token", { method: "POST" });
console.log(token.accessToken);

// Reconexão
const reconnectToken = await api.request("/pluggy/connect-token?item_id=abc123", {
  method: "POST",
});
```

---

#### `POST /pluggy/items` — Vincular Item Pluggy

Vincula um item do Pluggy ao usuário logado após conexão bem-sucedida no widget.

| Campo | Valor |
|-------|-------|
| **URL** | `POST /pluggy/items` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `item_id` (obrigatório) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `item_id` | `string` | ✅ | ID do item retornado pelo widget Pluggy Connect |

**Response `200 OK`:**
```json
{
  "id": "a1b2c3d4-pluggy-item-id",
  "connectorId": 201,
  "status": "UPDATED",
  "createdAt": "2026-04-02 19:00:00",
  "updatedAt": "2026-04-02 19:00:00"
}
```

**Insomnia:**
```
POST http://localhost:8000/pluggy/items?item_id=a1b2c3d4-pluggy-item-id

Headers:
  Authorization: Bearer <seu_token>
```

**React / TypeScript:**
```typescript
interface PluggyItemResponse {
  id: string;
  connectorId: number;
  status: string;
  createdAt: string;
  updatedAt: string;
}

async function linkPluggyItem(itemId: string) {
  const item = await api.post<PluggyItemResponse>(
    `/pluggy/items?item_id=${itemId}`
  );
  return item;
}
```

**Node.js:**
```javascript
async function linkPluggyItem(itemId) {
  return api.request(`/pluggy/items?item_id=${itemId}`, { method: "POST" });
}
```

---

#### `GET /pluggy/items` — Listar Items do Usuário

Retorna todos os items Pluggy vinculados ao usuário logado.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /pluggy/items` |
| **Auth** | ✅ Bearer Token |

**Response `200 OK`:**
```json
[
  {
    "id": 1,
    "pluggy_item_id": "a1b2c3d4-pluggy-item-id",
    "user_id": "uuid-do-usuario",
    "status": "UPDATED",
    "connector_id": 201,
    "created_at": "2026-04-02T19:00:00",
    "updated_at": "2026-04-02T19:00:00"
  }
]
```

**Insomnia:**
```
GET http://localhost:8000/pluggy/items

Headers:
  Authorization: Bearer <seu_token>
```

**React / TypeScript:**
```typescript
interface PluggyItem {
  id: number;
  pluggy_item_id: string;
  user_id: string;
  status: string;
  connector_id: number;
  created_at: string;
  updated_at: string;
}

const items = await api.get<PluggyItem[]>("/pluggy/items");
```

**Node.js:**
```javascript
const items = await api.request("/pluggy/items");
console.log(items);
```

---

#### `GET /pluggy/accounts` — Listar Contas Bancárias

Retorna todas as contas bancárias do usuário. Sincroniza com o Pluggy e persiste no banco local.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /pluggy/accounts` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `item_id` (opcional) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `item_id` | `integer` | ❌ | ID interno do item para filtrar contas |

**Response `200 OK`:**
```json
[
  {
    "id": "pluggy-account-uuid",
    "name": "Conta Corrente",
    "type": "BANK",
    "subtype": "CHECKING_ACCOUNT",
    "number": "12345-6",
    "balance": 5432.10,
    "currencyCode": "BRL"
  }
]
```

**Insomnia:**
```
GET http://localhost:8000/pluggy/accounts

Headers:
  Authorization: Bearer <seu_token>

-- OU filtrado por item: --
GET http://localhost:8000/pluggy/accounts?item_id=1
```

**React / TypeScript:**
```typescript
interface Account {
  id: string;
  name: string;
  type: string;
  subtype: string;
  number: string;
  balance: number;
  currencyCode: string;
}

// Todas as contas
const accounts = await api.get<Account[]>("/pluggy/accounts");

// Filtrado por item
const filteredAccounts = await api.get<Account[]>("/pluggy/accounts?item_id=1");
```

**Node.js:**
```javascript
const accounts = await api.request("/pluggy/accounts");
const filtered = await api.request("/pluggy/accounts?item_id=1");
```

---

#### `GET /pluggy/transactions` — Listar Transações

Retorna todas as transações do usuário. Sincroniza com o Pluggy e persiste no banco local.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /pluggy/transactions` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `account_id` (opcional) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `account_id` | `integer` | ❌ | ID interno da conta para filtrar transações |

**Response `200 OK`:**
```json
[
  {
    "id": "pluggy-tx-uuid",
    "description": "PIX RECEBIDO - JOAO",
    "amount": 1500.00,
    "date": "2026-04-01 00:00:00",
    "type": "CREDIT",
    "status": "POSTED"
  },
  {
    "id": "pluggy-tx-uuid-2",
    "description": "COMPRA CARTAO - SUPERMERCADO",
    "amount": -234.56,
    "date": "2026-03-30 00:00:00",
    "type": "DEBIT",
    "status": "POSTED"
  }
]
```

**Insomnia:**
```
GET http://localhost:8000/pluggy/transactions

Headers:
  Authorization: Bearer <seu_token>

-- OU filtrado por conta: --
GET http://localhost:8000/pluggy/transactions?account_id=5
```

**React / TypeScript:**
```typescript
interface Transaction {
  id: string;
  description: string;
  amount: number;
  date: string;
  type: string;
  status: string;
}

// Todas as transações
const transactions = await api.get<Transaction[]>("/pluggy/transactions");

// Filtradas por conta
const filtered = await api.get<Transaction[]>("/pluggy/transactions?account_id=5");
```

**Node.js:**
```javascript
const transactions = await api.request("/pluggy/transactions");
const filtered = await api.request("/pluggy/transactions?account_id=5");
```

---

#### `GET /pluggy/investments` — Listar Investimentos

Retorna todos os investimentos do usuário. Sincroniza com o Pluggy e persiste no banco local.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /pluggy/investments` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `item_id` (opcional) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `item_id` | `integer` | ❌ | ID interno do item para filtrar investimentos |

**Response `200 OK`:**
```json
[
  {
    "id": "pluggy-inv-uuid",
    "name": "CDB Banco XYZ",
    "number": "INV-001",
    "balance": 15000.00,
    "type": "FIXED_INCOME",
    "subtype": "CDB",
    "value": 15000.00,
    "quantity": 1.0
  }
]
```

**Insomnia:**
```
GET http://localhost:8000/pluggy/investments

Headers:
  Authorization: Bearer <seu_token>

-- OU filtrado: --
GET http://localhost:8000/pluggy/investments?item_id=1
```

**React / TypeScript:**
```typescript
interface Investment {
  id: string;
  name: string;
  number: string | null;
  balance: number;
  type: string;
  subtype: string | null;
  value: number | null;
  quantity: number | null;
}

const investments = await api.get<Investment[]>("/pluggy/investments");
const filtered = await api.get<Investment[]>("/pluggy/investments?item_id=1");
```

**Node.js:**
```javascript
const investments = await api.request("/pluggy/investments");
const filtered = await api.request("/pluggy/investments?item_id=1");
```

---

#### `GET /pluggy/loans` — Listar Empréstimos

Retorna todos os empréstimos/financiamentos do usuário. Sincroniza com o Pluggy e persiste no banco local.

| Campo | Valor |
|-------|-------|
| **URL** | `GET /pluggy/loans` |
| **Auth** | ✅ Bearer Token |
| **Query Params** | `item_id` (opcional) |

**Query Parameters:**

| Param | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `item_id` | `integer` | ❌ | ID interno do item para filtrar empréstimos |

**Response `200 OK`:**
```json
[
  {
    "id": "pluggy-loan-uuid",
    "name": "Financiamento Imobiliário",
    "amount": 350000.00,
    "outstandingBalance": 280000.00,
    "currencyCode": "BRL"
  }
]
```

**Insomnia:**
```
GET http://localhost:8000/pluggy/loans

Headers:
  Authorization: Bearer <seu_token>

-- OU filtrado: --
GET http://localhost:8000/pluggy/loans?item_id=1
```

**React / TypeScript:**
```typescript
interface Loan {
  id: string;
  name: string;
  amount: number;
  outstandingBalance: number | null;
  currencyCode: string;
}

const loans = await api.get<Loan[]>("/pluggy/loans");
const filtered = await api.get<Loan[]>("/pluggy/loans?item_id=1");
```

**Node.js:**
```javascript
const loans = await api.request("/pluggy/loans");
const filtered = await api.request("/pluggy/loans?item_id=1");
```

---

### 4. Webhooks

#### `POST /webhooks/pluggy` — Receber Webhook do Pluggy

Endpoint para receber notificações de eventos do Pluggy (ex: item atualizado, erro de login).

| Campo | Valor |
|-------|-------|
| **URL** | `POST /webhooks/pluggy` |
| **Auth** | ❌ Não (chamado pelo Pluggy) |
| **Content-Type** | `application/json` |

> [!NOTE]
> Este endpoint é chamado **automaticamente pelo Pluggy**, não pelo frontend. Configure a URL deste webhook no painel do Pluggy.

**Request Body (enviado pelo Pluggy):**
```json
{
  "event": "item/updated",
  "itemId": "a1b2c3d4-pluggy-item-id"
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `event` | `string` | Tipo do evento (ex: `item/updated`, `item/error`, `item/login_error`) |
| `itemId` | `string` | ID do item Pluggy associado ao evento |

**Response `200 OK`:**
```json
{
  "status": "received"
}
```

**Insomnia (simulação para testes):**
```
POST http://localhost:8000/webhooks/pluggy

Headers:
  Content-Type: application/json

Body (JSON):
{
  "event": "item/updated",
  "itemId": "a1b2c3d4-pluggy-item-id"
}
```

**Node.js (simulação):**
```javascript
async function simulateWebhook(event, itemId) {
  const response = await fetch("http://localhost:8000/webhooks/pluggy", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ event, itemId }),
  });
  return response.json();
}

await simulateWebhook("item/updated", "a1b2c3d4-pluggy-item-id");
```

---

### 5. Queries

#### `POST /queries/consolidated` — Busca Consolidada

Pesquisa todas as informações financeiras (contas, transações, investimentos, empréstimos) de um usuário pelo CPF/CNPJ. Requer confirmação de senha e gera log de auditoria.

| Campo | Valor |
|-------|-------|
| **URL** | `POST /queries/consolidated` |
| **Auth** | ✅ Bearer Token |
| **Content-Type** | `application/json` |

> [!CAUTION]
> Este é um endpoint sensível. Requer a **senha do usuário logado** para confirmar a operação. Toda tentativa de acesso é registrada no **Audit Log**.

**Request Body:**
```json
{
  "cpf": "12345678909",
  "current_password": "U2FsdGVkX1+...(senha criptografada AES)...",
  "start_date": "2026-01-01",
  "end_date": "2026-04-01"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `cpf` | `string` | ✅ | CPF/CNPJ do usuário a ser pesquisado |
| `current_password` | `string` | ✅ | Senha criptografada do usuário logado (confirmação) |
| `start_date` | `string (date)` | ❌ | Data inicial do filtro (formato `YYYY-MM-DD`) |
| `end_date` | `string (date)` | ❌ | Data final do filtro (formato `YYYY-MM-DD`) |

**Response `200 OK`:**
```json
{
  "targetUser": {
    "id": "uuid-do-usuario-alvo",
    "email": "alvo@email.com",
    "cpf": "12345678909"
  },
  "queryFilters": {
    "startDate": "2026-01-01",
    "endDate": "2026-04-01"
  },
  "accounts": [
    {
      "id": "pluggy-account-uuid",
      "name": "Conta Corrente",
      "type": "BANK",
      "subtype": "CHECKING_ACCOUNT",
      "balance": 5432.10,
      "currencyCode": "BRL"
    }
  ],
  "transactions": [
    {
      "id": "pluggy-tx-uuid",
      "description": "PIX RECEBIDO",
      "amount": 1500.00,
      "date": "2026-03-15 00:00:00",
      "type": "CREDIT",
      "status": "POSTED"
    }
  ],
  "investments": [
    {
      "id": "pluggy-inv-uuid",
      "name": "CDB Banco XYZ",
      "balance": 15000.00,
      "value": 15000.00,
      "quantity": 1.0,
      "type": "FIXED_INCOME"
    }
  ],
  "loans": [
    {
      "id": "pluggy-loan-uuid",
      "name": "Financiamento Imobiliário",
      "amount": 350000.00,
      "outstandingBalance": 280000.00,
      "currencyCode": "BRL"
    }
  ]
}
```

**Erros Possíveis:**

| Status | Detalhe |
|--------|---------|
| `401` | Could not validate credentials (token inválido) |
| `403` | Acesso negado: Senha inválida para realizar esta busca sensível |
| `404` | Usuário com este CPF/CNPJ não encontrado na base de dados |

**Insomnia:**
```
POST http://localhost:8000/queries/consolidated

Headers:
  Authorization: Bearer <seu_token>
  Content-Type: application/json

Body (JSON):
{
  "cpf": "12345678909",
  "current_password": "U2FsdGVkX1+abc123encryptedpassword==",
  "start_date": "2026-01-01",
  "end_date": "2026-04-01"
}
```

**React / TypeScript:**
```typescript
import CryptoJS from "crypto-js";

const INVITE_TOKEN = "meu_token_secreto_aws";

interface ConsolidatedResponse {
  targetUser: {
    id: string;
    email: string;
    cpf: string;
  };
  queryFilters: {
    startDate: string | null;
    endDate: string | null;
  };
  accounts: Account[];
  transactions: Transaction[];
  investments: Investment[];
  loans: Loan[];
}

async function searchConsolidated(
  cpf: string,
  password: string,
  startDate?: string,
  endDate?: string
): Promise<ConsolidatedResponse> {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const response = await api.post<ConsolidatedResponse>("/queries/consolidated", {
    cpf,
    current_password: encryptedPassword,
    start_date: startDate || null,
    end_date: endDate || null,
  });

  return response;
}

// Uso:
const data = await searchConsolidated(
  "12345678909",
  "minhaSenha123",
  "2026-01-01",
  "2026-04-01"
);

console.log(`Contas: ${data.accounts.length}`);
console.log(`Transações: ${data.transactions.length}`);
console.log(`Investimentos: ${data.investments.length}`);
console.log(`Empréstimos: ${data.loans.length}`);
```

**Node.js:**
```javascript
const CryptoJS = require("crypto-js");

const INVITE_TOKEN = "meu_token_secreto_aws";

async function searchConsolidated(cpf, password, startDate, endDate) {
  const encryptedPassword = CryptoJS.AES.encrypt(password, INVITE_TOKEN).toString();

  const response = await api.request("/queries/consolidated", {
    method: "POST",
    body: {
      cpf,
      current_password: encryptedPassword,
      start_date: startDate || null,
      end_date: endDate || null,
    },
  });

  return response;
}

// Uso:
const data = await searchConsolidated("12345678909", "senha123", "2026-01-01", "2026-04-01");
console.log(data.targetUser);
console.log(`Total de transações: ${data.transactions.length}`);
```

---

## Schemas

### User Schemas

```typescript
// UserCreate (Request)
interface UserCreate {
  email: string;       // Email válido
  cpf: string;         // CPF (11 dígitos) ou CNPJ (14 dígitos)
  password: string;    // Senha criptografada com AES
  invite_token: string; // Token de convite
}

// UserResponse (Response)
interface UserResponse {
  id: string;            // UUID v4
  email: string | null;
  cpf: string | null;
  created_at: string | null;
  updated_at: string | null;
}
```

### Token Schemas

```typescript
// Token (Response)
interface Token {
  access_token: string;
  token_type: string;  // sempre "bearer"
}

// TokenData (interno)
interface TokenData {
  email: string | null;
}
```

### Pluggy Schemas

```typescript
// ConnectTokenResponse
interface ConnectTokenResponse {
  accessToken: string;
}

// PluggyItemResponse
interface PluggyItemResponse {
  id: string;
  connectorId: number;
  status: string;    // "CREATED" | "UPDATING" | "UPDATED" | "LOGIN_ERROR"
  createdAt: string;
  updatedAt: string;
}

// Account
interface Account {
  id: string;
  name: string;
  type: string;       // "BANK" | "CREDIT"
  subtype: string;    // "CHECKING_ACCOUNT" | "SAVINGS_ACCOUNT" | "CREDIT_CARD"
  number: string;
  balance: number;
  currencyCode: string;
}

// Transaction
interface Transaction {
  id: string;
  description: string;
  amount: number;      // positivo = crédito, negativo = débito
  date: string;
  type: string;        // "CREDIT" | "DEBIT"
  status: string;      // "POSTED" | "PENDING"
}

// Investment
interface Investment {
  id: string;
  name: string;
  number: string | null;
  balance: number;
  type: string;        // "FIXED_INCOME" | "MUTUAL_FUND" | "EQUITY" | "UNKNOWN"
  subtype: string | null;
  value: number | null;
  quantity: number | null;
}

// Loan
interface Loan {
  id: string;
  name: string;
  amount: number;
  outstandingBalance: number | null;
  currencyCode: string;
}
```

### Search Schemas

```typescript
// ConsolidatedSearchRequest
interface ConsolidatedSearchRequest {
  cpf: string;                // CPF/CNPJ do alvo
  current_password: string;   // Senha criptografada do solicitante
  start_date?: string | null; // "YYYY-MM-DD"
  end_date?: string | null;   // "YYYY-MM-DD"
}

// ConsolidatedSearchResponse
interface ConsolidatedSearchResponse {
  targetUser: {
    id: string;
    email: string;
    cpf: string;
  };
  queryFilters: {
    startDate: string | null;
    endDate: string | null;
  };
  accounts: Account[];
  transactions: Transaction[];
  investments: Investment[];
  loans: Loan[];
}
```

---

## Tratamento de Erros

A API retorna erros no formato padrão do FastAPI:

```json
{
  "detail": "Mensagem de erro descritiva"
}
```

### Códigos de Status HTTP

| Código | Significado | Quando ocorre |
|--------|-------------|---------------|
| `200` | Sucesso | Requisição processada com sucesso |
| `400` | Bad Request | Dados inválidos, email/CPF duplicado, senha incorreta |
| `401` | Unauthorized | Token JWT ausente, expirado ou inválido |
| `403` | Forbidden | Token de convite inválido ou senha incorreta na busca |
| `404` | Not Found | Usuário/recurso não encontrado |
| `422` | Unprocessable Entity | Validação de dados falhou (CPF inválido, campos obrigatórios) |
| `500` | Internal Server Error | Erro interno (ex: falha na comunicação com Pluggy) |

### Exemplo de Tratamento em React/TypeScript

```typescript
try {
  const data = await api.get<Account[]>("/pluggy/accounts");
  setAccounts(data);
} catch (error) {
  if (error instanceof Error) {
    if (error.message.includes("401") || error.message.includes("credentials")) {
      // Token expirado - redirecionar para login
      localStorage.removeItem("access_token");
      router.push("/login");
    } else if (error.message.includes("403")) {
      // Sem permissão
      toast.error("Acesso negado.");
    } else {
      // Erro genérico
      toast.error(error.message);
    }
  }
}
```

### Exemplo de Tratamento em Node.js

```javascript
try {
  const data = await api.request("/pluggy/accounts");
  console.log(data);
} catch (error) {
  if (error.message.includes("401")) {
    console.error("Token expirado. Faça login novamente.");
    await api.login(email, password);
    // Retry
  } else {
    console.error("Erro:", error.message);
  }
}
```

---

## Resumo Rápido de Endpoints

| Método | Endpoint | Auth | Descrição |
|--------|----------|:----:|-----------|
| `GET` | `/` | ❌ | Health check simples |
| `GET` | `/health` | ❌ | Health check |
| `POST` | `/auth/register` | ❌ | Registrar novo usuário |
| `POST` | `/auth/login` | ❌ | Login (retorna JWT) |
| `GET` | `/auth/me` | ✅ | Dados do usuário logado |
| `POST` | `/pluggy/connect-token` | ✅ | Token para widget Pluggy |
| `POST` | `/pluggy/items` | ✅ | Vincular item Pluggy |
| `GET` | `/pluggy/items` | ✅ | Listar items do usuário |
| `GET` | `/pluggy/accounts` | ✅ | Listar contas bancárias |
| `GET` | `/pluggy/transactions` | ✅ | Listar transações |
| `GET` | `/pluggy/investments` | ✅ | Listar investimentos |
| `GET` | `/pluggy/loans` | ✅ | Listar empréstimos |
| `POST` | `/webhooks/pluggy` | ❌ | Receber webhook do Pluggy |
| `POST` | `/queries/consolidated` | ✅ | Busca consolidada com auditoria |

---

> **Documentação gerada em:** 02/04/2026  
> **Versão da API:** 1.0  
> **Framework:** FastAPI + SQLAlchemy + PostgreSQL
