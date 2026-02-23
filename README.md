# Bookio Backend - DevSecOps Pipeline

Educational project demonstrating DevSecOps with automated security pipeline on GitHub.

## Overview

- **Build**: Docker image creation (GitHub)
- **Security**: SAST (Semgrep), SCA (npm audit), Container Scan (Trivy), Secret Detection (Gitleaks) (GitHub)
- **Deploy**: Image deployment to GHCR (GitHub Container Registry) - triggered on PR merge only
- **DAST**: OWASP ZAP testing (GitHub)

## Quick Start

### Local Setup

```bash
npm install
npm start
```

App runs on `http://localhost:3000`

### GitHub Pipeline Flow

1. Create branch: `feature/name` or `bugfix/name`
2. Push to GitHub & open Pull Request:
   - `build.yml` → builds Docker image (without pushing to registry)
   - `security.yml` → SAST (Semgrep), SCA (npm audit), Container Scan (Trivy), Secret Detection (Gitleaks)
3. Code review & approval required
4. **PR merge to main** (only then):
   - `deploy.yml` → builds and pushes Docker image to `ghcr.io`
5. After deployment:
   - `dast.yml` → runs OWASP ZAP security tests

**Key**: Deploy **only happens after PR approval and merge** to main branch

## Project Structure

```
bookio-backend/
├── .github/workflows/       # CI/CD pipelines
│   ├── build.yml           # Build Docker image
│   ├── security.yml        # SAST, SCA, Container Scan
│   ├── deploy.yml          # Deploy image
│   └── dast.yml            # OWASP ZAP tests
│
├── config/
│   ├── semgrep.yml         # SAST rules
│   └── insomnia.json       # API tests
│
├── src/index.js            # Express app
├── Dockerfile              # Container definition
├── docker-compose.yml      # Local backend
├── docker-compose.dast.yml # DAST test environment
└── README.md
```

## Common Commands

```bash
# Development
npm start                           # Run app
npm install package-name           # Add dependency
npm audit                           # Check vulnerabilities

# Docker
docker build -t bookio:latest .    # Build image
docker run -p 3000:3000 bookio     # Run container
docker-compose up                  # Run with compose
docker-compose down                # Stop

# Git workflow
git checkout -b feature/name        # Create branch
git commit -m "feat: description"   # Commit
git push origin feature/name        # Push
# Create PR on GitHub
```

## Testing API

Use Insomnia collection: `config/insomnia.json`

### Manual Testing

```bash
# Start app
npm start

# Test endpoint
curl http://localhost:3000/livros
```

## GitHub Actions

### View Pipeline Results

1. Go to `Actions` tab
2. Select workflow run
3. Check job status and logs
4. Review `Security` → `Code scanning` for findings

### Troubleshooting Workflow Failures

1. Check Actions logs for error messages
2. Review security findings in Security tab
3. Fix issues locally
4. Push fix to trigger new pipeline run

## Security Tools

### SAST - Semgrep
- SQL injection detection
- Hardcoded secrets
- Weak cryptography
- Command injection

### Secret Detection - Gitleaks
- Detects exposed API keys
- Finds hardcoded credentials
- Scans entire git history
- Uploads findings to GitHub Security tab

### SCA - npm audit
- Dependency vulnerabilities
- Known CVEs

### Container Scan - Trivy
- Docker image vulnerabilities
- OS packages

### DAST - OWASP ZAP
- Runtime vulnerabilities
- XSS, CSRF, injection attacks

## Troubleshooting

### Docker: Cannot find module /app/index.js
The entry point is `src/index.js`. Make sure:
- Dockerfile has: `CMD ["node", "src/index.js"]`
- package.json has: `"start": "node src/index.js"`

### npm install fails
```bash
npm cache clean --force
rm -rf node_modules package-lock.json
npm install
```

### Port 3000 already in use
```bash
lsof -i :3000
kill -9 <PID>
```

### Docker permission denied
```bash
sudo usermod -aG docker $USER
# Logout and login
```

## Environment Variables

For local development, create a `.env` file (not committed to git):
```
# Secrets should be managed via GitHub Secrets, not hardcoded
MONGO_URI=mongodb://localhost:27017/bookio
NODE_ENV=development
```

---

**Status**: Production-ready DevSecOps pipeline with:
- ✅ PR-based gating (no deploy without approval)
- ✅ Automated security scanning (SAST, SCA, container, secrets)
- ✅ Secure artifact registry (GHCR)
- ✅ Correct Docker entrypoint configuration

