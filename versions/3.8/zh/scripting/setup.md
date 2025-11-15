#!/usr/bin/env bash
set -euo pipefail

REPO="https://github.com/ccccyy0924-pixel/3c-cy0924.git"
BRANCH="feature"
COMMIT_MSG="Add OpenAPI spec and mock server (docs/openapi.yaml + server/)"

# 可改為你本機已有的 working dir：若你已 clone repo，請把下面改為直接在該 repo 執行
WORKDIR="$(mktemp -d)"
echo "Using temp dir: $WORKDIR"
cd "$WORKDIR"

echo "Cloning repo..."
git clone "$REPO"
cd 3c-cy0924 || (echo "Failed to enter repo folder. 如果倉庫是 private，請先 clone 到本機使用有權限的帳號。" && exit 1)

echo "Creating new branch: $BRANCH"
git checkout -b "$BRANCH"

echo "Creating directories..."
mkdir -p docs server

echo "Writing docs/openapi.yaml..."
cat > docs/openapi.yaml <<'YAML'
openapi: 3.0.3
info:
  title: 三國主公 — 網頁遊戲 API
  description: OpenAPI 規範（最小可行產品 / 核心端點）
  version: "1.0.0"
servers:
  - url: http://localhost:3000
    description: Local dev server
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    Error:
      type: object
      properties:
        message:
          type: string
    User:
      type: object
      properties:
        id:
          type: integer
        username:
          type: string
        level:
          type: integer
        vip_level:
          type: integer
    AuthResponse:
      type: object
      properties:
        token:
          type: string
        user:
          $ref: '#/components/schemas/User'
    City:
      type: object
      properties:
        id:
          type: integer
        owner_id:
          type: integer
        name:
          type: string
        level:
          type: integer
        x:
          type: integer
        y:
          type: integer
    Resources:
      type: object
      properties:
        city_id:
          type: integer
        food:
          type: integer
        wood:
          type: integer
        iron:
          type: integer
        coin:
          type: integer
    General:
      type: object
      properties:
        id:
          type: integer
        owner_id:
          type: integer
        name:
          type: string
        rarity:
          type: string
        level:
          type: integer
        lead:
          type: integer
        force:
          type: integer
        intellect:
          type: integer
        skills:
          type: array
          items:
            type: object
    Troop:
      type: object
      properties:
        id:
          type: integer
        owner_id:
          type: integer
        city_id:
          type: integer
        general_id:
          type: integer
        count:
          type: integer
        troop_type:
          type: string
        march_state:
          type: string
    Alliance:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        leader_id:
          type: integer
        members_count:
          type: integer
    Battle:
      type: object
      properties:
        id:
          type: integer
        attacker_id:
          type: integer
        defender_id:
          type: integer
        result:
          type: string
        log:
          type: object
paths:
  /auth/register:
    post:
      summary: 註冊新用戶
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                username:
                  type: string
                password:
                  type: string
      responses:
        "201":
          description: 註冊成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        "400":
          description: 參數錯誤
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /auth/login:
    post:
      summary: 登入並取得 JWT
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                username:
                  type: string
                password:
                  type: string
      responses:
        "200":
          description: 登入成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        "401":
          description: 認證失敗
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /users/me:
    get:
      summary: 取得當前使用者資訊
      security:
        - bearerAuth: []
      responses:
        "200":
          description: 使用者資訊
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        "401":
          description: 未授權
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /cities/{cityId}:
    get:
      summary: 取得城池資訊
      parameters:
        - in: path
          name: cityId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: 城池資訊
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/City'
        "404":
          description: 未找到
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /cities/{cityId}/resources:
    get:
      summary: 取得城池資源狀態
      parameters:
        - in: path
          name: cityId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: 資源資訊
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Resources'
  /cities/{cityId}/collect-resources:
    post:
      summary: 收取資源 (領取產出)
      security:
        - bearerAuth: []
      parameters:
        - in: path
          name: cityId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: 收取成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Resources'
        "401":
          description: 未授權
  /generals:
    get:
      summary: 列出玩家武將（或公共列表）
      security:
        - bearerAuth: []
      responses:
        "200":
          description: 武將列表
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/General'
  /generals/recruit:
    post:
      summary: 招募武將
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                type:
                  type: string
                  description: 招募類型（普通/高級/活動）
      responses:
        "201":
          description: 招募成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/General'
  /troops/march:
    post:
      summary: 發起行軍 / 出征
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                city_id:
                  type: integer
                target:
                  type: object
                  properties:
                    x:
                      type: integer
                    y:
                      type: integer
                    type:
                      type: string
                troops:
                  type: array
                  items:
                    type: object
                    properties:
                      general_id:
                        type: integer
                      count:
                        type: integer
      responses:
        "202":
          description: 出征已派出（進行中）
          content:
            application/json:
              schema:
                type: object
                properties:
                  march_id:
                    type: integer
                  eta_seconds:
                    type: integer
  /battles/{battleId}:
    get:
      summary: 查詢戰鬥/戰報
      parameters:
        - in: path
          name: battleId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: 戰鬥詳情
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Battle'
  /map/points:
    get:
      summary: 查詢世界地圖上的點（城池/資源點/要塞）
      parameters:
        - in: query
          name: bbox
          description: 查詢框 (x1,y1,x2,y2)
          required: false
          schema:
            type: string
      responses:
        "200":
          description: 地圖點列表
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:
                      type: integer
                    type:
                      type: string
                    x:
                      type: integer
                    y:
                      type: integer
                    owner:
                      type: string
  /alliances:
    get:
      summary: 列出同盟
      responses:
        "200":
          description: 同盟列表
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Alliance'
    post:
      summary: 創建同盟
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
      responses:
        "201":
          description: 同盟創建成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Alliance'
  /alliances/{allianceId}/join:
    post:
      summary: 加入同盟（申請/直接加入視權限）
      security:
        - bearerAuth: []
      parameters:
        - in: path
          name: allianceId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: 加入成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Alliance'
  /tasks:
    get:
      summary: 查詢任務（新手/日常）
      security:
        - bearerAuth: []
      responses:
        "200":
          description: 任務列表
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:
                      type: integer
                    title:
                      type: string
                    status:
                      type: string
security:
  - bearerAuth: []
YAML

echo "Writing server/package.json..."
cat > server/package.json <<'JSON'
{
  "name": "sanguo-master-api-mock",
  "version": "1.0.0",
  "description": "Mock backend for 三國主公 OpenAPI demo",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
JSON

echo "Writing server/index.js..."
cat > server/index.js <<'JS'
const express = require('express');
const bodyParser = require('body-parser');
const jwt = require('jsonwebtoken');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const PORT = process.env.PORT || 3000;
const JWT_SECRET = process.env.JWT_SECRET || 'replace_with_strong_secret';

// --- In-memory mock DB ---
let users = [];
let cities = [
  { id: 1, owner_id: 1, name: "許昌", level: 5, x: 100, y: 200 },
];
let resources = {
  1: { city_id: 1, food: 1000, wood: 800, iron: 500, coin: 1200 }
};
let generals = [
  { id: 1, owner_id: 1, name: "關羽", rarity: "epic", level: 10, lead: 90, force: 95, intellect: 60, skills: [] }
];
let alliances = [];
let marches = [];
let battles = [];

// --- Helpers ---
function authMiddleware(req, res, next) {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ message: 'Missing Authorization' });
  const parts = auth.split(' ');
  if (parts.length !== 2) return res.status(401).json({ message: 'Invalid Authorization' });
  const token = parts[1];
  try {
    const payload = jwt.verify(token, JWT_SECRET);
    req.user = payload;
    next();
  } catch (e) {
    return res.status(401).json({ message: 'Invalid token' });
  }
}

// --- Routes ---
// Auth
app.post('/auth/register', (req, res) => {
  const { username, password } = req.body;
  if (!username || !password) return res.status(400).json({ message: 'username/password required' });
  if (users.find(u => u.username === username)) return res.status(400).json({ message: 'username exists' });
  const user = { id: users.length + 1, username, level: 1, vip_level: 0 };
  users.push({ ...user, password });
  // create default city for user
  const cityId = cities.length + 1;
  cities.push({ id: cityId, owner_id: user.id, name: `${username}的城池`, level: 1, x: 10 + cityId, y: 10 + cityId });
  resources[cityId] = { city_id: cityId, food: 100, wood: 100, iron: 50, coin: 200 };
  const token = jwt.sign({ id: user.id, username: user.username }, JWT_SECRET);
  res.status(201).json({ token, user });
});

app.post('/auth/login', (req, res) => {
  const { username, password } = req.body;
  const user = users.find(u => u.username === username && u.password === password);
  if (!user) return res.status(401).json({ message: 'Invalid credentials' });
  const token = jwt.sign({ id: user.id, username: user.username }, JWT_SECRET);
  res.json({ token, user: { id: user.id, username: user.username, level: user.level, vip_level: user.vip_level } });
});

// Get current user
app.get('/users/me', authMiddleware, (req, res) => {
  const user = users.find(u => u.id === req.user.id);
  if (!user) return res.status(404).json({ message: 'User not found' });
  res.json({ id: user.id, username: user.username, level: user.level, vip_level: user.vip_level });
});

// Cities
app.get('/cities/:cityId', (req, res) => {
  const city = cities.find(c => c.id === Number(req.params.cityId));
  if (!city) return res.status(404).json({ message: 'City not found' });
  res.json(city);
});
app.get('/cities/:cityId/resources', (req, res) => {
  const r = resources[Number(req.params.cityId)];
  if (!r) return res.status(404).json({ message: 'Resources not found' });
  res.json(r);
});
app.post('/cities/:cityId/collect-resources', authMiddleware, (req, res) => {
  const cityId = Number(req.params.cityId);
  const r = resources[cityId];
  if (!r) return res.status(404).json({ message: 'Resources not found' });
  // mock: zero out resources and return previous
  const collected = { ...r };
  resources[cityId] = { city_id: cityId, food: 0, wood: 0, iron: 0, coin: 0 };
  res.json(collected);
});

// Generals
app.get('/generals', authMiddleware, (req, res) => {
  const list = generals.filter(g => g.owner_id === req.user.id || g.owner_id === 1);
  res.json(list);
});
app.post('/generals/recruit', authMiddleware, (req, res) => {
  const id = generals.length + 1;
  const g = { id, owner_id: req.user.id, name: `新武將${id}`, rarity: 'common', level: 1, lead: 10, force: 10, intellect: 10, skills: [] };
  generals.push(g);
  res.status(201).json(g);
});

// Troops / March
app.post('/troops/march', authMiddleware, (req, res) => {
  const { city_id, target, troops } = req.body;
  const marchId = marches.length + 1;
  const eta_seconds = 60; // mock
  marches.push({ marchId, owner_id: req.user.id, city_id, target, troops, eta_seconds });
  res.status(202).json({ march_id: marchId, eta_seconds });
});

// Battles
app.get('/battles/:battleId', (req, res) => {
  const b = battles.find(x => x.id === Number(req.params.battleId));
  if (!b) return res.status(404).json({ message: 'Battle not found' });
  res.json(b);
});

// Map
app.get('/map/points', (req, res) => {
  const pts = cities.map(c => ({ id: c.id, type: 'city', x: c.x, y: c.y, owner: c.owner_id ? `user${c.owner_id}` : null }));
  res.json(pts);
});

// Alliances
app.get('/alliances', (req, res) => res.json(alliances));
app.post('/alliances', authMiddleware, (req, res) => {
  const id = alliances.length + 1;
  const a = { id, name: req.body.name || `盟${id}`, leader_id: req.user.id, members_count: 1 };
  alliances.push(a);
  res.status(201).json(a);
});
app.post('/alliances/:allianceId/join', authMiddleware, (req, res) => {
  const a = alliances.find(x => x.id === Number(req.params.allianceId));
  if (!a) return res.status(404).json({ message: 'Alliance not found' });
  a.members_count += 1;
  res.json(a);
});

// Tasks
app.get('/tasks', authMiddleware, (req, res) => {
  res.json([{ id: 1, title: '新手任務：建造農田', status: 'open' }]);
});

app.listen(PORT, () => {
  console.log(`Mock API running on http://localhost:${PORT}`);
});
JS

echo "Writing README.md..."
cat > README.md <<'MD'
# 三國主公 — OpenAPI 規範與 Mock Server

此目錄包含本專案的 OpenAPI 規範與一個簡易的本地 mock 後端，供前端與後端在開發初期對齊 API contract 與本地測試使用。

目錄結構（建議）
- docs/openapi.yaml        # OpenAPI 3.0 規範
- server/package.json      # mock server 依賴與 scripts
- server/index.js          # mock server 實作（Express + JWT, in-memory）

快速上手（本機）
1. 取得 feature 分支（或在本機建立並切到 feature）
   git fetch origin feature
   git checkout feature

2. 在專案根目錄（已包含 docs/ 與 server/）安裝依賴並啟動 mock server
   ```bash
   cd server
   npm install
   # 設定 JWT secret
   export JWT_SECRET="your_local_secret"   # macOS / Linux
   # Windows PowerShell:
   # $env:JWT_SECRET="your_local_secret"
   npm start
