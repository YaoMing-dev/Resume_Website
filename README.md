# Karaa — Resume Builder

<p align="center">
  <img src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Node.js-Express-339933?logo=node.js&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/MongoDB-Mongoose-47A248?logo=mongodb&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Redis-Cache-DC382D?logo=redis&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square" />
</p>

A full-stack web application for creating, customizing, and exporting professional resumes. Built with React and Node.js, featuring OAuth authentication, real-time preview, drag-and-drop editing, PDF/DOCX export, and public resume sharing.

---

## Features

**Resume Editor**
- Multi-section form: personal info, experience, education, skills (with proficiency levels), projects, certifications, and activities
- Real-time preview as you type, with template switching without data loss
- Drag-and-drop section reordering via [@dnd-kit](https://dndkit.com)
- Full customization panel: font family, color scheme, layout (single-column, two-column, timeline, grid, etc.), photo style and position, spacing

**Templates**
- Multiple professionally designed templates across 4 categories: Modern, Professional, Creative, Minimalist
- Filter, sort, and compare templates side by side before selecting
- ATS-friendly template options

**Export & Sharing**
- Export to PDF (client-side via html2canvas + jsPDF, or server-side via Puppeteer for consistent output)
- Export to DOCX (server-side)
- Generate a public share link with optional password protection, expiry date, and download toggle
- Track view counts on shared resumes

**Authentication**
- Email/password registration with email verification flow
- OAuth 2.0 via Google and LinkedIn (Passport.js)
- Auto account linking when the same email is used across providers
- Guest mode: create and export a resume without registering, with data migration on sign-up

**Version Control**
- Save resume snapshots at any point
- Restore any previous version
- Compare two versions side by side

**Other**
- Notification system
- Bilingual UI (English / Vietnamese)
- Fully containerized with Docker Compose

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Frontend** | React 18, Vite, React Router DOM, @dnd-kit, html2canvas, jsPDF |
| **Backend** | Node.js, Express.js, Mongoose (MongoDB), Passport.js |
| **Database** | MongoDB 7 (primary), Redis 7 (caching & guest sessions) |
| **Auth** | JWT (access + refresh tokens), Google OAuth 2.0, LinkedIn OAuth |
| **Security** | bcryptjs, Helmet.js, express-rate-limit, express-validator, CORS |
| **Export** | html2canvas + jsPDF (client), Puppeteer (server), docx (server) |
| **DevOps** | Docker, Docker Compose, Nginx (frontend serving) |

---

## Architecture Overview

```
Karaa/
├── frontend/               # React SPA (Vite)
│   └── src/
│       ├── pages/          # Landing, Auth, Dashboard, Editor, Profile, Share, ...
│       ├── components/     # ResumePreview, EditableField, SortableSection, ...
│       ├── context/        # AuthContext, LanguageContext
│       ├── hooks/          # useArrayCRUD (generic CRUD for resume sections)
│       └── services/       # resumeService, templateService, authService, guestService
│
├── backend/                # Express REST API
│   └── src/
│       ├── models/         # User, Resume, Template, Notification, Guide
│       ├── controllers/    # authController, resumeController, templateController, ...
│       ├── routes/         # Versioned REST routes (/api/v1/*)
│       ├── middleware/      # auth (protect, authorize, optionalAuth), validation, errorHandler
│       ├── config/         # MongoDB, Redis, Passport strategies
│       └── utils/          # JWT utils, asyncHandler, email service
│
├── docker-compose.yml      # Orchestrates MongoDB, Redis, Backend, Frontend
└── .env.example
```

**Data flow:** React SPA → REST API (`/api/v1`) → MongoDB (persisted data) / Redis (cached responses, guest sessions)

**Caching strategy:** Redis with layered TTLs — 5 min for resume lists, 30 min for profiles, 1 hr for templates, 24 hr for template categories. Cache is invalidated on writes.

---

## Database Models

**User** — name, email, hashed password, avatar, phone, location, bio, role (user/admin), googleId, linkedinId, provider, isEmailVerified, resetPasswordToken, lastLogin, soft delete (deletedAt)

**Resume** — user ref, template ref, title, full content JSON (personal, experience, education, skills, projects, certifications, activities), customization config, sharing settings (shareId, isPublic, password, expiresAt, viewCount), version history, soft delete

**Template** — name, category, layout config (type, columns), typography, color palette, photo config, section ordering, features flags (atsFriendly, multiPage, hasPhoto, hasCharts), popularity score

**Notification** — user ref, type, title, message, isRead, readAt

**Guide** — Career guides by industry (tech, healthcare, finance, etc.) with tips, dos/don'ts, recommended templates

---

## API Reference

<details>
<summary>Auth — <code>/api/v1/auth</code></summary>

```
POST   /register               Register with email + send verification
POST   /login                  Email/password login
POST   /verify-email           Verify email with code
POST   /resend-verification    Resend verification email
POST   /forgotpassword         Initiate password reset
PUT    /resetpassword/:token   Complete password reset
PUT    /updatepassword         Change password (protected)
POST   /logout                 Logout
GET    /me                     Current user (protected)
GET    /google                 Google OAuth
GET    /google/callback        Google OAuth callback
GET    /linkedin               LinkedIn OAuth
GET    /linkedin/callback      LinkedIn OAuth callback
```
</details>

<details>
<summary>Resumes — <code>/api/v1/resumes</code></summary>

```
GET    /                       List resumes (paginated, sortable, searchable)
POST   /                       Create resume
GET    /:id                    Get resume
PUT    /:id                    Update resume
DELETE /:id                    Soft delete
POST   /:id/duplicate          Clone resume
GET    /share/:shareId         Get public shared resume (no auth)
POST   /:id/share              Generate share link
PUT    /:id/share              Update share settings (password, expiry, download)
POST   /:id/version            Save version snapshot
GET    /:id/versions           Get version history
POST   /:id/restore/:version   Restore version
GET    /:id/compare/:v1/:v2    Compare two versions
GET    /:id/export/docx        Export as Word document
GET    /:id/export/pdf         Export as PDF (server-side)
POST   /:id/export/pdf-html    Export from HTML string (Puppeteer)
GET    /stats                  Resume statistics
```
</details>

<details>
<summary>Templates, Users, Notifications, Guest — <code>/api/v1/*</code></summary>

```
# Templates
GET    /templates              List templates (with caching)
GET    /templates/categories   Get categories
GET    /templates/popular      Get popular templates
GET    /templates/:id          Get template
POST   /templates              Create (admin only)
PUT    /templates/:id          Update (admin only)
DELETE /templates/:id          Delete (admin only)

# Users
GET    /users/profile          Get profile
PUT    /users/profile          Update profile
PUT    /users/change-password  Change password
DELETE /users/account          Soft delete account
DELETE /users/account/permanent Permanent deletion (GDPR)
POST   /users/avatar           Upload avatar
GET    /users/activity         Get activity log

# Notifications
GET    /notifications               List notifications
GET    /notifications/unread-count  Unread count
PUT    /notifications/:id/read      Mark as read
PUT    /notifications/read-all      Mark all as read
DELETE /notifications/:id           Delete

# Guest
POST   /guest/session          Create guest session
POST   /guest/resume           Save guest resume (Redis TTL)
GET    /guest/resume/:id       Get guest resume
PUT    /guest/resume/:id       Update guest resume
GET    /guest/resumes          List guest resumes
DELETE /guest/resume/:id       Delete guest resume
POST   /guest/migrate          Migrate guest data to user account
```
</details>

---

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/get-started) and Docker Compose, **or** Node.js 18+ with MongoDB and Redis installed locally

### Run with Docker (recommended)

```bash
git clone https://github.com/YaoMing-dev/Karaa.git
cd Karaa

cp .env.example .env
# Edit .env: set JWT secrets, Google/LinkedIn OAuth credentials, SMTP config

docker-compose up -d --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:5000/api/v1 |
| Health check | http://localhost:5000/api/v1/health |

### Run locally (development)

```bash
# Backend
cd backend && npm install
cp .env.example .env   # fill in your values
npm run dev

# Frontend (new terminal)
cd frontend && npm install
npm run dev            # http://localhost:5173
```

### Environment variables

```env
# Server
NODE_ENV=development
PORT=5000

# Database
MONGODB_URI=mongodb://localhost:27017/resume_builder
REDIS_HOST=localhost
REDIS_PORT=6379

# Auth secrets (generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))")
JWT_SECRET=
JWT_REFRESH_SECRET=
SESSION_SECRET=
ENCRYPTION_SECRET=

# Google OAuth — https://console.cloud.google.com
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=http://localhost:5000/api/v1/auth/google/callback

# LinkedIn OAuth — https://www.linkedin.com/developers/apps
LINKEDIN_CLIENT_ID=
LINKEDIN_CLIENT_SECRET=
LINKEDIN_CALLBACK_URL=http://localhost:5000/api/v1/auth/linkedin/callback

# Email (for verification and password reset)
SMTP_HOST=
SMTP_PORT=587
SMTP_EMAIL=
SMTP_PASSWORD=

# Frontend
FRONTEND_URL=http://localhost:5173
```

---

## Security

- Passwords hashed with **bcryptjs** (salt rounds 10)
- JWT access tokens (7d) + refresh tokens (30d), HttpOnly cookies in production
- OAuth 2.0 with auto account linking across providers
- Email verification required on registration
- Sensitive resume fields encrypted at rest
- Rate limiting: 100 requests per 15 minutes
- Helmet.js security headers, CORS whitelist, input validation via express-validator
- Soft deletes preserve data; permanent deletion available for GDPR compliance

---

## License

[MIT](LICENSE) © [YaoMing-dev](https://github.com/YaoMing-dev)
