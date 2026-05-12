# Next.js Development Server Troubleshooting Guide

## Overview
This guide provides a systematic approach to diagnose and resolve issues where the Next.js/Vite development server hangs on startup or times out waiting for ports to be ready.

---

## Phase 1: Initial Diagnostics

### 1.1 Identify the Symptom
- [ ] Server logs show "Starting dev server"
- [ ] Process hangs indefinitely without error messages
- [ ] Timeout error after X seconds
- [ ] Process exits with code (note the code)
- [ ] Port appears to be in use but no process running

### 1.2 Gather Current State Information
```bash
# Check all Node/Vite processes
ps aux | grep -E "vite|node|pnpm" | grep -v grep

# Check port usage (replace XXXX with port number)
lsof -i :XXXX
netstat -tlnp | grep XXXX

# View recent error logs
tail -100 /vercel/share/v0-project/logs/* 2>/dev/null

# Check npm process list
npm list -g 2>/dev/null | head -20
```

---

## Phase 2: Port Conflict Detection

### 2.1 Identify All Required Ports
For a monorepo with multiple Vite apps, document expected ports:
```
mockup-sandbox:   5173 (or fallback)
powerlvl:         5174 (or fallback)
api-server:       5175 (or fallback)
```

### 2.2 Check Port Availability
```bash
# List all listening ports
lsof -i -P -n | grep LISTEN

# Check specific ports
for port in 5173 5174 5175; do
  echo "=== Port $port ===" 
  lsof -i :$port || echo "Port $port is free"
done

# Check if ports are stuck in TIME_WAIT state
netstat -tlnp | grep TIME_WAIT
```

### 2.3 Kill Stale Processes
```bash
# Find and kill old dev processes
pkill -f "pnpm run dev"
pkill -f "pnpm run start"
pkill -f "vite"
pkill -f "node"

# Wait for processes to fully terminate
sleep 3

# Verify processes are gone
ps aux | grep -E "vite|node|pnpm" | grep -v grep
```

### 2.4 Check strictPort Configuration
```bash
# Critical: strictPort setting prevents fallback to alternative ports
grep -r "strictPort" artifacts/*/vite.config.ts

# Should be: strictPort: false (allows port fallback)
# NOT: strictPort: true (forces exact port, fails if taken)
```

---

## Phase 3: Verify Server Logs for Errors

### 3.1 Read Debug Logs
```bash
# In v0 environment, read the debug log file
cat user_read_only_context/v0_debug_logs.log | tail -200

# Look for specific errors:
# - "PORT environment variable is required"
# - "BASE_PATH environment variable is required"
# - "EADDRINUSE: Address already in use"
# - "Cannot find module"
# - "SyntaxError"
```

### 3.2 Check Individual Service Logs
```bash
# Test each service individually
cd /vercel/share/v0-project/artifacts/mockup-sandbox
pnpm run dev 2>&1 | head -50

cd /vercel/share/v0-project/artifacts/powerlvl
pnpm run dev 2>&1 | head -50

cd /vercel/share/v0-project/artifacts/api-server
npm run start 2>&1 | head -50  # or pnpm run dev
```

### 3.3 Common Error Messages
| Error | Cause | Solution |
|-------|-------|----------|
| `PORT environment variable is required` | Missing PORT default value | Add default: `process.env.PORT \|\| "5173"` |
| `BASE_PATH environment variable is required` | Missing BASE_PATH default value | Add default: `process.env.BASE_PATH \|\| "/"` |
| `EADDRINUSE: Address already in use :::5173` | Port already in use | Kill stale processes or set `strictPort: false` |
| `Cannot find module 'X'` | Dependencies not installed | Run `pnpm install` in workspace root |
| `dist file not found` | Source changes not compiled | Rebuild: `pnpm run build` in affected package |

---

## Phase 4: Dependency Verification

### 4.1 Check Installation Status
```bash
# Verify pnpm workspaces are set up
ls -la /vercel/share/v0-project/pnpm-workspace.yaml

# Check node_modules installation
ls -la /vercel/share/v0-project/node_modules | head -20

# Count installed packages
find /vercel/share/v0-project/node_modules -maxdepth 1 -type d | wc -l
```

### 4.2 Rebuild Dependencies
```bash
# Clean install in workspace
cd /vercel/share/v0-project
rm -f pnpm-lock.yaml
pnpm install

# Or in individual package (if isolated issue)
cd /vercel/share/v0-project/artifacts/powerlvl
pnpm install
```

### 4.3 Verify Dist Files are Current
```bash
# Check if dist files exist
ls -la /vercel/share/v0-project/artifacts/api-server/dist/

# If dist is stale (modified before source), rebuild
cd /vercel/share/v0-project/artifacts/api-server
find dist -type f -delete && rmdir dist  # Clear dist
pnpm run build                           # Rebuild from source
```

---

## Phase 5: Process Blocking & Startup Issues

### 5.1 Identify Process Blocking
```bash
# Check if specific process is stuck
strace -p <PID> 2>&1 | head -50

# Monitor system calls in real-time while starting dev server
strace -e trace=network -f pnpm dev 2>&1 | head -100
```

### 5.2 Common Blocking Causes
- **Synchronous file I/O**: Check for large file reads during startup
- **Network calls**: DNS lookups, external API calls during initialization
- **Module resolution**: Circular dependencies, missing files
- **Build process hanging**: TypeScript compilation taking too long

### 5.3 Check for Circular Dependencies
```bash
# If using TypeScript
cd /vercel/share/v0-project
pnpm exec tsc --noEmit --listFilesOnly 2>&1 | grep -i "error\|circular"

# Check vite config for syntax errors
node -c artifacts/mockup-sandbox/vite.config.ts
node -c artifacts/powerlvl/vite.config.ts
```

---

## Phase 6: Environment Configuration Validation

### 6.1 Verify Environment Variables
```bash
# Check if env files exist
ls -la /vercel/share/.env.* 2>/dev/null

# View available env vars (for dev server)
env | grep -E "PORT|BASE_PATH|NODE_ENV" | sort

# Manually set defaults for testing
export PORT=5173
export BASE_PATH=/
export NODE_ENV=development
```

### 6.2 Validate Vite Config
```bash
# Check vite.config.ts for required fields
grep -E "base:|port|strictPort|host" artifacts/*/vite.config.ts

# Correct pattern should be:
# - port: process.env.PORT || "5173"
# - base: process.env.BASE_PATH || "/"
# - strictPort: false
# - host: "0.0.0.0"
```

### 6.3 Check Build Configuration
```bash
# Verify next.config.js or vite config doesn't have blocking options
cat artifacts/powerlvl/vite.config.ts | grep -A5 -B5 "optimizeDeps\|ssr"

# Look for inline plugins that might hang
grep -n "plugin" artifacts/*/vite.config.ts
```

---

## Phase 7: Systematic Troubleshooting Flow

```
START
  |
  v
[1] Kill all stale processes
  |
  v
[2] Check port availability (5173, 5174, 5175)
  | If ports in use:
  |   -> Check strictPort setting
  |   -> Set strictPort: false
  |
  v
[3] Read debug logs for specific errors
  | If PORT/BASE_PATH error:
  |   -> Add default values to config
  | If "Cannot find module":
  |   -> Run pnpm install
  | If "dist not found":
  |   -> Rebuild affected package
  |
  v
[4] Test individual services
  | Run: pnpm run dev in each artifacts/* folder
  | Wait 10+ seconds for startup
  |
  v
[5] If still hanging:
  |   -> Check for synchronous I/O in startup
  |   -> Check for network calls on initialization
  |   -> Review recent code changes (git diff)
  |
  v
[6] Clear and rebuild everything
  |   -> pnpm install (fresh deps)
  |   -> pnpm run build (rebuild all dist)
  |   -> pnpm dev (start fresh)
  |
  v
SUCCESS: All services listening on ports
```

---

## Phase 8: Quick Reference Commands

### Clean Start
```bash
cd /vercel/share/v0-project
pkill -f "pnpm run dev"
sleep 2
pnpm install
pnpm run build
pnpm dev
```

### Verify All Services
```bash
# Wait 10 seconds for startup
sleep 10

# Check all ports are listening
lsof -i -P -n | grep LISTEN | grep -E "5173|5174|5175"

# Test connectivity
curl http://localhost:5173 2>/dev/null | head -20
curl http://localhost:5174 2>/dev/null | head -20
curl http://localhost:5175/api/health 2>/dev/null || echo "API server"
```

### Debug Single Service
```bash
cd /vercel/share/v0-project/artifacts/powerlvl
PORT=5174 BASE_PATH=/ pnpm run dev --debug 2>&1
```

---

## Prevention Checklist

- [ ] Ensure all vite.config.ts files have `PORT || "default"` fallbacks
- [ ] Set `strictPort: false` in dev server configs
- [ ] Add default values for BASE_PATH environment variable
- [ ] Keep dist files in sync with source (rebuild after changes)
- [ ] Document expected ports for each service
- [ ] Add `.env.example` with required variables
- [ ] Use `pnpm run build` before testing in production
- [ ] Monitor for port conflicts before starting dev server
- [ ] Keep dependencies updated: `pnpm up`

---

## When to Escalate

If after following all steps the server still hangs:
1. Check git history for recent changes that broke startup
2. Test with a fresh clone of the repository
3. Verify Node.js version compatibility
4. Check system resources (disk space, memory, CPU)
5. Review firewall/network rules blocking ports
