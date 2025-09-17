# Immersia

# ImmersiaRP — Spécification complète

**Stack choisi :**

* **Frontend** : React (TypeScript) + Vite + Tailwind CSS
* **Backend** : Node.js (TypeScript) avec **NestJS** ou **Express + Fastify** (exemples fournis pour Express + TypeScript)
* **Base de données** : PostgreSQL (avec Prisma comme ORM)
* **Realtime** : WebSocket via `socket.io` (ou `ws`) pour le chat et actions en temps réel
* **Authentification** : OAuth2 (Google, Discord) + JWT pour sessions
* **Stockage fichiers** : S3-compatible (MinIO pour dev) pour avatars, images
* **Ia** : Optionnel — OpenAI / modèle local pour PNJ dynamiques
* **Déploiement** : Docker Compose / Kubernetes pour production

---

## Objectifs fonctionnels

1. Connexion via Google/Discord, création et soumission de personnage.
2. Validation manuelle des personnages par admin/modo.
3. Environnements (salles) multi-textuelles : RP full écrit.
4. Commandes d’action structurées (ex: `/attaque`, `/parle`).
5. Gestion d’inventaire, compétences, réputation.
6. Modération robuste : rapports, logs, sanction.
7. Interface admin / modérateur pour valider persos, gérer joueurs et événements.
8. Option IA : génération de PNJ, quêtes, événements.

---

## Architecture globale

* **Client (React)**

  * Pages : Login, Création perso, Dashboard, Salle RP, Profil, Admin
  * WebSocket pour échange en temps réel
  * REST pour opérations CRUD (personnage, environnements, fichiers)

* **API (Node + TypeScript)**

  * Auth routes (OAuth, token refresh)
  * Personnage routes (POST /characters, GET /characters/\:id)
  * Environnement routes (GET /rooms, POST /rooms)
  * Modération routes (GET /reports, POST /ban)
  * WebSocket server (événements chat, actions, notifications)

* **DB (Postgres)**

  * Tables : users, characters, rooms, messages, items, reports, bans, events

* **Système de stockage** : S3 pour avatars & exports

---

## Schéma de base de données (Prisma / SQL)

### Modèle Prisma (simplifié)

```prisma
generator client { provider = "prisma-client-js" }

datasource db { provider = "postgresql" url = env("DATABASE_URL") }

model User {
  id         String    @id @default(uuid())
  email      String    @unique
  name       String?
  oauthId    String?   @unique
  provider   String?   // google, discord
  role       Role      @default(USER)
  createdAt  DateTime  @default(now())
  characters Character[]
}

model Character {
  id          String   @id @default(uuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  displayName String
  bio         String?
  traits      Json     // {charisme: 3, furtivite: 2}
  avatarUrl   String?
  status      CharStatus @default(PENDING)
  createdAt   DateTime @default(now())
}

enum CharStatus { PENDING APPROVED REJECTED }

enum Role { USER MODERATOR ADMIN }

model Room {
  id          String   @id @default(uuid())
  slug        String   @unique
  title       String
  description String?
  rules       String?
  isPublic    Boolean  @default(true)
  createdAt   DateTime @default(now())
}

model Message {
  id         String   @id @default(uuid())
  roomId     String
  room       Room     @relation(fields: [roomId], references: [id])
  characterId String?
  content    String
  type       MessageType @default(ROLEPLAY) // ROLEPLAY, OOC, SYSTEM
  meta       Json?
  createdAt  DateTime @default(now())
}

enum MessageType { ROLEPLAY OOC SYSTEM }

model Report {
  id         String   @id @default(uuid())
  reporterId String
  targetId   String // could be userId or messageId depending
  reason     String
  status     ReportStatus @default(OPEN)
  createdAt  DateTime @default(now())
}

enum ReportStatus { OPEN IN_PROGRESS RESOLVED }
```

---

## Routes API (REST) — exemples principaux

**Auth**

* `GET /auth/google` -> redirection OAuth
* `GET /auth/google/callback` -> échange code -> create/login user -> JWT
* `POST /auth/refresh` -> refresh token

**Utilisateur / Perso**

* `POST /characters` -> soumettre perso (status = PENDING)
* `GET /characters/me` -> récupère persos utilisateur
* `GET /characters/:id` -> voir fiche (si approved ou owner)
* `PATCH /characters/:id` -> modification (re-soumission si change)

**Rooms / RP**

* `GET /rooms` -> liste des lieux
* `GET /rooms/:slug` -> détails
* `POST /rooms/:slug/join` -> rejoindre (enregistre présence)

**Admin / Modération** (RBAC : role moderator/admin)

* `GET /admin/characters?status=PENDING` -> liste des persos à valider
* `POST /admin/characters/:id/approve` -> approuver perso
* `POST /admin/reports/:id/resolve` -> traiter un rapport
* `POST /admin/ban` -> bannir utilisateur

---

## WebSocket (Socket.IO) — Événements clés

**Noms d’événements (convention)** : `client:event` / `server:event`

**Client -> Serveur**

* `client:joinRoom` `{ roomId, characterId, token }`
* `client:leaveRoom` `{ roomId, characterId }`
* `client:sendMessage` `{ roomId, characterId, content, type, meta }`
* `client:action` `{ roomId, characterId, action: '/attaque', targetId, payload }`

**Serveur -> Client**

* `server:message` `{ message }` (broadcast dans la room)
* `server:actionResult` `{ success, description, meta }`
* `server:systemNotice` `{ text }`
* `server:presenceUpdate` `{ roomId, players[] }`

**Logique serveur**

* À la réception d'une action, appliquer logique métier (ex: calcul de réussite) puis broadcast d’un `server:actionResult` + éventuellement générer un `Message` si action narrative.

---

## Exemple de code : Auth Google (Express + TypeScript)

```ts
// auth.ts (Express)
import express from 'express';
import fetch from 'node-fetch';
import jwt from 'jsonwebtoken';

const router = express.Router();

router.get('/google/callback', async (req, res) => {
  const code = req.query.code as string;
  // échange code -> tokens
  const tokenRes = await fetch('https://oauth2.googleapis.com/token', { method: 'POST', body: new URLSearchParams({
    code,
    client_id: process.env.GOOGLE_CLIENT_ID!,
    client_secret: process.env.GOOGLE_CLIENT_SECRET!,
    redirect_uri: process.env.GOOGLE_REDIRECT_URI!,
    grant_type: 'authorization_code'
  })});
  const tokenJson = await tokenRes.json();
  const idToken = tokenJson.id_token;
  // décoder idToken ou appeler userinfo
  const userInfoRes = await fetch('https://www.googleapis.com/oauth2/v3/userinfo', {
    headers: { Authorization: `Bearer ${tokenJson.access_token}` }
  });
  const profile = await userInfoRes.json();

  // Ici : find or create user dans la DB
  // Générer JWT et rediriger le client
  const jwtToken = jwt.sign({ sub: profile.sub, email: profile.email }, process.env.JWT_SECRET!);
  res.redirect(`${process.env.CLIENT_URL}/auth/success?token=${jwtToken}`);
});

export default router;
```

---

## Exemple de code : WebSocket minimal (socket.io)

```ts
import { createServer } from 'http';
import { Server } from 'socket.io';
import express from 'express';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: '*' } });

io.on('connection', (socket) => {
  console.log('client connected', socket.id);

  socket.on('client:joinRoom', async (payload) => {
    const { roomId, characterId } = payload;
    socket.join(roomId);
    // récupérer présence, notifier
    io.to(roomId).emit('server:presenceUpdate', { roomId, players: [] });
  });

  socket.on('client:sendMessage', async (payload) => {
    // validation, stockage en DB
    const message = {
      id: 'tmp', roomId: payload.roomId, content: payload.content, createdAt: new Date()
    };
    io.to(payload.roomId).emit('server:message', { message });
  });
});

httpServer.listen(3000);
```

---

## Exemple de composant React (TypeScript) : Chat RP

```tsx
import React, { useEffect, useState } from 'react';
import { io } from 'socket.io-client';

const socket = io(import.meta.env.VITE_WS_URL as string);

export function RoomChat({ roomId, character }) {
  const [messages, setMessages] = useState<any[]>([]);
  const [text, setText] = useState('');

  useEffect(() => {
    socket.emit('client:joinRoom', { roomId, characterId: character.id });
    socket.on('server:message', (data) => setMessages((m) => [...m, data.message]));
    return () => { socket.emit('client:leaveRoom', { roomId, characterId: character.id }); };
  }, [roomId]);

  const send = () => {
    if (!text.trim()) return;
    socket.emit('client:sendMessage', { roomId, characterId: character.id, content: text, type: 'ROLEPLAY' });
    setText('');
  };

  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-auto">
        {messages.map((m) => <div key={m.id} className="p-2">{m.content}</div>)}
      </div>
      <div className="p-2 flex">
        <input value={text} onChange={e => setText(e.target.value)} className="flex-1" />
        <button onClick={send} className="ml-2 p-2 rounded bg-blue-600">Envoyer</button>
      </div>
    </div>
  );
}
```

---

## Validation des personnages (Admin flow)

1. L'utilisateur soumet son personnage (`status = PENDING`).
2. Les modos voient une file `GET /admin/characters?status=PENDING` et ouvrent la fiche.
3. Ils peuvent : `Approve`, `Request changes` (renvoie une note à l'utilisateur) ou `Reject`.
4. Actions : notification au user, journal des décisions, raisons stockées pour audit.

**UI Admin** : liste filtrable, aperçu du background, boutons d’action rapide, champ notes privées.

---

## Modération et sécurité

* RBAC (Role-based access control) centralisé sur middleware.
* Rate-limiting pour endpoints critiques.
* Filtrage des messages : blacklist/regex + IA pour détection de harcèlement.
* Système de reports avec étiquettes et priorités.
* Historique complet (logs immuables) pour sanctions.

---

## Système d’actions / mécanique de réussite

* **Approche simple** : actions lancées avec `traits` + RNG (ex: `roll = random(1..20) + traitValue`) ; seuils définis par l’environnement.
* **Approche narrative** : renvoyer un résultat textuel riche (succès critique, succès partiel, échec) et modifier l’état (pv, réputation, items).
* Stocker les actions importantes comme `Message` de `type: SYSTEM` pour l’historique.

---

## IA : génération de PNJ / quêtes (prompt template)

> **Prompt master pour génération de PNJ**

```
Génère un PNJ pour un univers fantasy urbain. Format JSON:
{
  "name": "",
  "role": "merchant|guard|oracle|mystic|thief",
  "background": "brève biographie (2-3 phrases)",
  "motivation": "ce qu'il veut",
  "dialogue_samples": ["réplique1", "réplique2"],
  "hooks": ["idée de quête courte", "objet recherché"]
}

Contrainte: garder le texte court, utilisable directement pour insertion en DB.
```

---

## Frontend — structure de dossier (suggestion)

```
client/
├─ public/
├─ src/
│  ├─ components/
│  ├─ pages/
│  ├─ hooks/
│  ├─ services/ (api clients, socket)
│  ├─ styles/
│  └─ main.tsx
```

## Backend — structure de dossier (suggestion)

```
server/
├─ src/
│  ├─ controllers/
│  ├─ services/
│  ├─ websockets/
│  ├─ prisma/
│  ├─ auth/
│  └─ index.ts
├─ prisma/schema.prisma
```

---

## Docker Compose (dev)

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: immersia_dev
    volumes:
      - db-data:/var/lib/postgresql/data
  minio:
    image: minio/minio
    command: server /data
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    ports:
      - "9000:9000"
  api:
    build: ./server
    env_file: .env.dev
    depends_on: [db, minio]
    ports: ["3000:3000"]
  client:
    build: ./client
    ports: ["5173:5173"]
volumes:
  db-data:
```

---

## Tests & CI

* Tests unitaires backend : Jest + Supertest
* Tests frontend : Vitest + React Testing Library
* CI pipeline : lint -> tests -> build -> docker image -> push

---

## Export / sauvegarde d'aventures

* Export JSON ou PDF (générer via `puppeteer` ou `pdfkit`) du journal d'aventure d'un personnage.
* Endpoint admin pour export global d'une room

---

## Option P2P — considérations

Faire une app P2P (peer-to-peer) pour RP en texte est possible (ex: WebRTC, lib p2p comme libp2p) mais :

* **Avantages** : moins de coûts serveurs, résilience.
* **Inconvénients** : authentification + modération compliquées, stockage d’historique difficile, sécurité et sanctions hard à implémenter, découverte des pairs.

Suggestion : version hybride — serveurs centraux pour auth/modération/stockage, échanges P2P facultatifs entre joueurs pour features non critiques.

---

## Suggestions de features futures

* Système de maps visuelles et repères de position en temps réel
* Intégration vocale optionnelle (voice channels) pour events
* Mini-scripting pour PNJ (behaviour trees)
* Plugins communautaires pour ajouter règles / systèmes de jeu

---

## Checklist minimale pour MVP (fonctionnel)

* Auth Google/Discord + JWT
* Création & soumission de personnage (PENDING)
* Interface admin pour approuver persos
* Liste de rooms et chat texte en temps réel (socket.io)
* Système de reports & logs
* Déploiement Docker Compose local

---

## Ressources & librairies recommandées

* Prisma (ORM) — [https://www.prisma.io/](https://www.prisma.io/)
* Socket.IO — [https://socket.io/](https://socket.io/)
* React + Vite — [https://vitejs.dev/](https://vitejs.dev/)
* Tailwind CSS — [https://tailwindcss.com/](https://tailwindcss.com/)
* Passport.js / `openid-client` pour OAuth
* OpenAI API (optionnel) pour génération PNJ & quêtes

---
