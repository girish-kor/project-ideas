# INSTALLATION.md — N2N Toolz Setup Guide

---

## System Requirements

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Windows 10, macOS 11, Ubuntu 20.04 | Windows 11, macOS 13+, Ubuntu 22.04 |
| Node.js | 18.x LTS | 20.x LTS |
| npm | 9.x | 10.x |
| RAM | 4 GB | 8 GB+ |
| Disk | 500 MB | 2 GB (plugins + projects) |
| Display | 1280×720 | 1920×1080 |

---

## Installation Methods

### Method 1: From Source (Developer Mode)

```bash
# 1. Clone the repository
git clone https://github.com/your-org/n2n-toolz.git
cd n2n-toolz

# 2. Install dependencies
npm install

# 3. Start in development mode
npm run dev
```

### Method 2: Build Desktop Binary

```bash
# Build for current OS
npm run build:electron

# Build for specific targets
npm run build:win      # Windows .exe
npm run build:mac      # macOS .dmg
npm run build:linux    # Linux .AppImage / .deb
```

Output → `releases/` directory.

### Method 3: Pre-built Release

Download from [Releases Page](https://github.com/your-org/n2n-toolz/releases)

| Platform | File |
|---|---|
| Windows | `n2n-toolz-setup-x.x.x.exe` |
| macOS | `n2n-toolz-x.x.x.dmg` |
| Linux | `n2n-toolz-x.x.x.AppImage` |

---

## Environment Setup

### 1. Node.js Version Manager (Recommended)

```bash
# Using nvm
nvm install 20
nvm use 20

# Using fnm
fnm install 20
fnm use 20
```

### 2. Install Global Dependencies (Optional for Dev)

```bash
npm install -g prettier eslint typescript
```

---

## First Launch

1. Open N2N Toolz
2. On welcome screen → **Create New Project**
3. Choose project directory
4. A blank canvas opens with default workflow tab
5. Drag a node from Node Palette → canvas
6. Press **F5** to run

---

## Directory Initialization

On first run, N2N Toolz creates:

```
~/n2n-toolz-workspace/
├── projects/           → Your .n2n project files
├── plugins/            → Installed plugin packages
├── exports/            → Generated Node.js code
└── logs/               → App + execution logs
```

Override default workspace path: `Settings → General → Default project directory`

---

## Plugin Registry Setup (Optional)

```bash
# Set custom registry in Settings > Plugins > Plugin Registry URL
# Default: https://registry.n2ntoolz.io

# Install a plugin manually
npm install my-n2n-plugin --prefix ~/.n2n-toolz/plugins
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| App won't start | Check Node.js version: `node -v` must be ≥ 18 |
| Canvas blank | Try `View → Fit to Screen` or restart |
| Plugin fails to load | Check plugin exports `n2n.type = "node-plugin"` in `package.json` |
| Code export fails | Ensure output directory has write permissions |
| Electron build fails | Run `npm run clean` then `npm install` |

---

## Uninstall

```bash
# Remove app
rm -rf /path/to/n2n-toolz

# Remove workspace data (DESTRUCTIVE — deletes projects)
rm -rf ~/n2n-toolz-workspace
```
