# Multi-Stage Dockerfile: Complete Line-by-Line Explanation

## 🎯 The Complete Dockerfile

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

---

## 🏗️ STAGE 1: BUILD STAGE (Lines 1-6)

### Line 1: `FROM node:18-alpine AS build`

```dockerfile
FROM node:18-alpine AS build
     ↑            ↑      ↑
     |            |      └─ Give this stage a NAME "build"
     |            └──────── Use Alpine version (smaller)
     └───────────────────── Start with Node.js version 18
```

**What it does:**
- Downloads the `node:18-alpine` image (~150 MB)
- This image contains: Node.js, npm, and minimal Alpine Linux
- **Names this stage "build"** so we can reference it later

**Why "AS build"?**
- We'll have multiple stages (multiple FROM statements)
- Naming helps us copy files from this stage later
- Think of it like: `Stage1 = build stage`

**Visual:**
```
┌─────────────────────────────────────┐
│  Stage Name: "build"                │
│  Base Image: node:18-alpine         │
│  Size: ~150 MB                      │
│  Contains: Node.js 18 + npm + Alpine│
└─────────────────────────────────────┘
```

---

### Line 2: `WORKDIR /app`

```dockerfile
WORKDIR /app
        ↑
        └─ Creates /app directory and enters it
```

**What it does:**
- Creates a directory `/app` inside the container
- Sets it as the current working directory
- All subsequent commands run from `/app`

**Equivalent to:**
```bash
mkdir -p /app
cd /app
```

**Current state:**
```
Container filesystem:
/
├── bin/
├── etc/
├── usr/
└── app/  ← We are HERE
```

---

### Line 3: `COPY package*.json ./`

```dockerfile
COPY package*.json ./
     ↑            ↑
     |            └─ Destination: ./ (current dir = /app)
     └────────────── Source: Your computer
```

**What it does:**
- Copies `package.json` and `package-lock.json` from your computer
- Pastes them into `/app/` in the container

**Why copy package files FIRST, before other code?**
- **Docker Layer Caching!** 🎯
- If package.json hasn't changed, Docker reuses the cached `npm install` layer
- This makes rebuilds MUCH faster

**Your computer:**
```
my-project/
├── package.json        ← Copy THIS
├── package-lock.json   ← Copy THIS
├── server.js           ← NOT YET!
└── src/                ← NOT YET!
```

**Container after this line:**
```
/app/
├── package.json        ← Copied
└── package-lock.json   ← Copied
```

---

### Line 4: `RUN npm ci --only=production`

```dockerfile
RUN npm ci --only=production
    ↑      ↑                 ↑
    |      |                 └─ Install ONLY production dependencies
    |      └───────────────────── "Clean Install" - faster, more reliable
    └──────────────────────────── Execute command in container
```

**What it does:**
- Runs `npm ci` (clean install) command
- Installs dependencies from `package-lock.json`
- **Only installs production dependencies** (skips devDependencies)

**What's the difference?**

| Command | Speed | Uses | Dependencies |
|---------|-------|------|--------------|
| `npm install` | Slower | package.json | All (dev + prod) |
| `npm ci` | Faster | package-lock.json | All (dev + prod) |
| `npm ci --only=production` | Fastest | package-lock.json | **Production only** |

**What gets installed:**

**package.json:**
```json
{
  "dependencies": {
    "express": "^4.18.2"        ← INSTALLED ✅
  },
  "devDependencies": {
    "webpack": "^5.75.0",       ← SKIPPED ❌
    "babel": "^7.20.0",         ← SKIPPED ❌
    "jest": "^29.3.0"           ← SKIPPED ❌
  }
}
```

**Why skip devDependencies?**
- We don't need testing tools (jest) in production
- We don't need build tools (webpack) in final image
- Smaller image = faster deployment

**Container after this line:**
```
/app/
├── package.json
├── package-lock.json
└── node_modules/           ← NEW! Dependencies installed
    ├── express/
    └── ... (production only)
```

---

### Line 5: `COPY . .`

```dockerfile
COPY . .
     ↑ ↑
     | └─ Destination: ./ (/app)
     └─── Source: Everything from your project
```

**What it does:**
- Copies **ALL remaining files** from your project
- Source code, configuration, everything!

**What gets copied:**

**Your computer:**
```
my-project/
├── package.json          (already copied)
├── package-lock.json     (already copied)
├── server.js             ← Copy NOW
├── src/                  ← Copy NOW
│   ├── index.js
│   └── components/
├── webpack.config.js     ← Copy NOW
└── README.md             ← Copy NOW
```

**Container after this line:**
```
/app/
├── package.json
├── package-lock.json
├── node_modules/
├── server.js             ← NEW!
├── src/                  ← NEW!
│   ├── index.js
│   └── components/
├── webpack.config.js     ← NEW!
└── README.md             ← NEW!
```

**Why copy package.json FIRST, then code LATER?**

**Caching magic:**
```
Build #1: (First time)
- COPY package.json      [New - Execute]
- RUN npm ci            [New - Execute] ← Takes 2 minutes
- COPY . .              [New - Execute]
- RUN npm run build     [New - Execute]

Build #2: (Changed server.js only)
- COPY package.json      [Cached ✅]
- RUN npm ci            [Cached ✅] ← Instant! No re-install!
- COPY . .              [New - Execute] ← Only this runs
- RUN npm run build     [New - Execute]
```

**Without this optimization:**
```
Build #2: (Changed server.js)
- COPY . .              [New - package.json changed date!]
- RUN npm ci            [New - Re-run] ← Wastes 2 minutes!
```

---

### Line 6: `RUN npm run build`

```dockerfile
RUN npm run build
    ↑          ↑
    |          └─ Runs the "build" script from package.json
    └──────────── Execute command
```

**What it does:**
- Runs the build script defined in package.json
- Typically: Webpack/Vite/Rollup bundles and optimizes your code
- Creates production-ready files

**What happens during build:**

**package.json:**
```json
{
  "scripts": {
    "build": "webpack --mode production"
  }
}
```

**Build process:**
```
Source Code (Development):
src/
├── index.jsx           (React JSX - not runnable)
├── App.tsx             (TypeScript - not runnable)
├── styles.scss         (SASS - not runnable)
└── components/         (100+ files)

        ↓ BUILD ↓
    (Compile, Bundle, Minify, Optimize)
        
Built Code (Production):
dist/
├── bundle.js           (Single file, minified)
├── styles.css          (Compiled CSS)
└── index.html          (Ready to serve)
```

**Container after this line:**
```
/app/
├── package.json
├── node_modules/
├── src/                (Source code)
└── dist/               ← NEW! Built files
    ├── bundle.js
    ├── styles.css
    └── index.html
```

**Stage 1 Complete!** This stage now contains:
- ✅ Source code
- ✅ node_modules
- ✅ Built files (/app/dist/)
- ✅ Build tools
- ❌ **Size: ~300-500 MB**

---

## 🎁 STAGE 2: PRODUCTION STAGE (Lines 8-11)

### Line 8: `FROM node:18-alpine`

```dockerfile
FROM node:18-alpine
     ↑
     └─ Start COMPLETELY FRESH!
```

**CRITICAL UNDERSTANDING:**
- This starts a **brand new, empty container**
- Everything from Stage 1 is **GONE** (except what we explicitly copy)
- It's like starting over with a clean slate

**Visual:**
```
STAGE 1 (build):                    STAGE 2 (production):
┌──────────────────────┐            ┌──────────────────────┐
│ Source code  300 MB  │            │                      │
│ node_modules 150 MB  │   ──────>  │   FRESH START!       │
│ Build tools  100 MB  │   DISCARD  │   Empty Alpine       │
│ Built dist/   50 MB  │            │   ~40 MB             │
└──────────────────────┘            └──────────────────────┘
     600 MB TOTAL                         40 MB (will add more)
```

**No name on this FROM:**
- Stage 1: `FROM node:18-alpine AS build` ← Named
- Stage 2: `FROM node:18-alpine` ← Unnamed (final stage)
- The last unnamed stage becomes your final image

---

### Line 9: `WORKDIR /app`

```dockerfile
WORKDIR /app
```

**What it does:**
- Creates `/app` directory in this **NEW** container
- Remember: This is a completely fresh container!

---

### Line 10: `COPY --from=build /app/dist ./dist`

```dockerfile
COPY --from=build /app/dist ./dist
     ↑            ↑         ↑
     |            |         └─ Destination: /app/dist (current stage)
     |            └─────────── Source: /app/dist (from build stage)
     └──────────────────────── Copy FROM the "build" stage
```

**This is the MAGIC LINE! 🎩✨**

**What it does:**
- Reaches back to Stage 1 (the "build" stage)
- Grabs ONLY the `/app/dist/` folder
- Copies it to the current stage

**Visual:**
```
STAGE 1 ("build"):              STAGE 2 (current):
┌──────────────────────┐        ┌──────────────────────┐
│ /app/                │        │ /app/                │
│ ├── src/             │        │ └── dist/  ← Copied! │
│ ├── node_modules/    │        │     ├── bundle.js    │
│ ├── dist/            │ ━━━━━> │     └── styles.css   │
│ │   ├── bundle.js    │  COPY  │                      │
│ │   └── styles.css   │  ONLY  │                      │
│ ├── webpack.config   │  THIS! │ Everything else      │
│ └── package.json     │        │ is LEFT BEHIND!      │
└──────────────────────┘        └──────────────────────┘
```

**What does NOT get copied:**
- ❌ Source code (`src/`)
- ❌ Build tools (`webpack.config.js`)
- ❌ Tests
- ❌ Documentation
- ✅ **Only production-ready files!**

---

### Line 11: `COPY --from=build /app/node_modules ./node_modules`

```dockerfile
COPY --from=build /app/node_modules ./node_modules
     ↑            ↑                 ↑
     |            |                 └─ Destination in current stage
     |            └───────────────────Source from build stage
     └────────────────────────────────Copy from named stage
```

**What it does:**
- Copies the `node_modules/` folder from Stage 1
- **But wait!** These are production dependencies only (remember `--only=production`?)

**Visual:**
```
STAGE 1:                        STAGE 2:
┌──────────────────────┐        ┌──────────────────────┐
│ /app/node_modules/   │        │ /app/                │
│ ├── express/         │ ━━━━━> │ ├── dist/            │
│ ├── lodash/          │  COPY  │ └── node_modules/    │
│ └── ... (prod only)  │        │     ├── express/     │
└──────────────────────┘        │     └── lodash/      │
                                └──────────────────────┘
```

**Why copy node_modules?**
- Your app needs runtime dependencies (like express)
- These are already installed and compiled in Stage 1
- No need to run `npm install` again!

---

### Line 12: `CMD ["node", "dist/server.js"]`

```dockerfile
CMD ["node", "dist/server.js"]
    ↑      ↑               ↑
    |      |               └─ Path to your app's entry point
    |      └─────────────────Command to run
    └────────────────────────Default command when container starts
```

**What it does:**
- Sets the default command to run when container starts
- Runs Node.js with your built application

**Final container structure:**
```
/app/
├── dist/
│   ├── bundle.js       ← Your compiled app
│   └── styles.css
└── node_modules/
    ├── express/        ← Runtime dependencies
    └── ...
```

**When you run the container:**
```powershell
docker run myapp

# Executes:
node dist/server.js
```

---

## 📊 Size Comparison: Stage 1 vs Stage 2

### Stage 1 (Build) - DISCARDED
```
Base image (node:18-alpine):  150 MB
Source code (src/):            10 MB
node_modules (all):           200 MB
Build tools:                   50 MB
Built files (dist/):           20 MB
TOTAL:                        430 MB ← THROWN AWAY!
```

### Stage 2 (Production) - FINAL IMAGE
```
Base image (node:18-alpine):  150 MB
Built files (dist/):           20 MB
node_modules (prod only):      80 MB
TOTAL:                        250 MB ← THIS IS YOUR IMAGE!
```

**Savings: 180 MB (42% smaller!)**

---

## 🎯 Complete Flow Visualization

```
YOUR COMPUTER                 STAGE 1 (build)              STAGE 2 (production)
┌──────────────┐             ┌──────────────┐             ┌──────────────┐
│ package.json │────────────>│ package.json │             │              │
│ src/         │             │ src/         │             │              │
│ server.js    │────────────>│ server.js    │             │              │
└──────────────┘             │              │             │              │
                             │ npm ci       │             │              │
                             │ npm build    │             │              │
                             │              │             │              │
                             │ Creates:     │──────┐      │              │
                             │ └── dist/    │      │      │              │
                             │     bundle.js│      ├─────>│ dist/        │
                             └──────────────┘      │      │   bundle.js  │
                                                   │      │              │
                             ┌──────────────┐      │      │              │
                             │ node_modules │      └─────>│ node_modules │
                             │ (production) │             │ (production) │
                             └──────────────┘             └──────────────┘
                                   ↓                             ↓
                              DISCARDED                    FINAL IMAGE
                             (Not in image)                 (250 MB)
```

---

## 💡 Why This Pattern Works

### Traditional Single-Stage (Bad):
```
Everything goes in → Everything stays → Big image
Source + Tools + Build + Output = 1 GB
```

### Multi-Stage (Good):
```
Everything goes in → Build happens → Extract only output
Source + Tools + Build (Stage 1) ──> Output only (Stage 2)
430 MB (temporary) ──────────────> 250 MB (final)
```

---

## 🧪 Try It Yourself

### Create a Test Project

```powershell
# Create project
mkdir multistage-test
cd multistage-test

# Create package.json
@"
{
  "name": "test",
  "scripts": {
    "build": "echo 'Building...' && mkdir -p dist && echo 'console.log(\"Hello\")' > dist/server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
"@ | Out-File package.json -Encoding UTF8

# Create Dockerfile
@"
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
"@ | Out-File Dockerfile -Encoding UTF8

# Build
docker build -t test-multistage .

# Check size
docker images test-multistage
```

---

## 🎓 Key Takeaways

1. **Two stages = Two FROM statements**
   - Stage 1: Build everything
   - Stage 2: Copy only what you need

2. **AS build = Naming a stage**
   - Allows you to reference it later with `--from=build`

3. **COPY order matters for caching**
   - package.json first → npm install → code last

4. **--from=build is the magic**
   - Reaches back to previous stage
   - Extracts only specific files

5. **Final image = Last FROM statement**
   - Everything before is temporary
   - Only the last stage becomes your image

---

## ❓ Common Questions

**Q: What happens to Stage 1?**
A: Completely discarded after build! It's temporary.

**Q: Can I have more than 2 stages?**
A: Yes! You can have 3, 4, 5... stages. Each FROM starts a new one.

**Q: Do I need to name every stage?**
A: Only if you need to copy FROM it. The last stage usually doesn't need a name.

**Q: Why not just delete files in Stage 1?**
A: Docker layers are immutable. Once added, they increase size even if deleted later.

---

Want me to show you a 3-stage build? Or help you create one for your specific project? 🚀s