# Server Docker Compose

Production deployment configuration for running frontend, backend, webhook, and n8n pipeline containers with queue mode.

## Architecture

Flow: `frontend → cloudflare → backend → webhook → n8n queue mode`

Components:
- Frontend (submodule)
- Backend (submodule)
- n8n-main (UI/API)
- n8n-webhook (webhook processing)
- n8n-worker (job processing)
- PostgreSQL (n8n database)
- Redis (queue backend)
- OrcaSlicer (3D model processing)

---

## Git Submodules

This repository uses git submodules to manage frontend and backend repositories. Submodules allow version pinning and reproducible deployments.

### Initial Setup (First Time Clone)

```bash
# Clone this repo 
git clone <this-repo-url>
cd server-docker-compose

# Init and setup for submodules 
git submodule update --init --recursive
```

### Adding a New Submodule (done once during repo setup)

```bash
# Add frontend submodule
git submodule add <frontend-repo-url> frontend

# Add backend submodule
git submodule add <backend-repo-url> backend

# Pin to specific version (recommended for production)
cd frontend
git checkout v1.0.0  # or specific commit/tag
cd ../backend
git checkout v1.0.0
cd ..

# Commit the submodule references
git add .gitmodules frontend backend
git commit -m "Add frontend and backend submodules at v1.0.0"
git push
```

### Daily Workflow

#### Pulling Latest Changes (from others)

```bash
# Pull parent repo changes
git pull

# Update submodules to match the commit refs in parent repo
git submodule update --init --recursive
```

#### Making Changes Inside a Submodule

```bash
# Navigate to submodule
cd frontend  # or backend

# Make your changes
# ... edit files ...

# Commit and push to the submodule's repo
git add .
git commit -m "Your change description"
git push origin main  # or your branch name

# Go back to parent repo
cd ..

# Update parent repo to point to new commit
git add frontend
git commit -m "Update frontend submodule: your change description"
git push
```

#### Updating a Submodule to a New Version

```bash
# Navigate to submodule
cd frontend

# Fetch latest changes
git fetch

# Checkout desired version (tag, branch, or commit)
git checkout v1.2.0  # or main, or specific commit SHA

# Go back to parent repo
cd ..

# Commit the submodule update
git add frontend
git commit -m "Update frontend to v1.2.0"
git push
```

#### Checking Submodule Status

```bash
# See which commit each submodule is on
git submodule status

# See if submodules have uncommitted changes
git status

# See submodule remote URLs
git submodule
```

### Best Practices

#### 1. Version Pinning (Critical for Production)
- **Always pin to specific tags/releases** for production deployments
- Use semantic versioning (v1.0.0, v1.1.0, etc.) in your frontend/backend repos
- Avoid pointing to `main` or `master` branches in production

```bash
# Good (production)
cd frontend && git checkout v1.2.0

# Bad (production)
cd frontend && git checkout main
```

#### 2. Deployment Workflow
```bash
# On your development machine
cd frontend
git checkout v1.3.0  # new stable release
cd ..
git add frontend
git commit -m "Deploy frontend v1.3.0 to production"
git push

# On the server
git pull
git submodule update --init --recursive
docker-compose up -d --build
```

#### 3. Rollback Strategy
```bash
# Parent repo maintains version history
git log --oneline  # find previous deployment

# Rollback to previous version
git checkout <previous-commit>
git submodule update --init --recursive
docker-compose up -d --build

# Or rollback just one submodule
cd frontend
git checkout v1.2.0  # previous version
cd ..
git add frontend
git commit -m "Rollback frontend to v1.2.0"
```

#### 4. Never Commit Secrets
- Use `.env` files for environment-specific configuration
- Add `.env` to `.gitignore`
- Commit `.env.example` with placeholder values
- Document required environment variables

#### 5. Documentation
- Document which submodule versions are deployed
- Keep a CHANGELOG.md for deployment history
- Tag parent repo commits for production releases

```bash
# After deploying a new version
git tag -a prod-2024-01-23 -m "Production deployment: frontend v1.3.0, backend v2.1.0"
git push --tags
```

#### 6. Consistent Submodule Updates
```bash
# Add this to your deployment script
git pull && git submodule update --init --recursive
```

#### 7. Avoid Detached HEAD Issues
Submodules are often in "detached HEAD" state (pointing to specific commit). This is normal and expected for production. If you need to make changes:

```bash
cd frontend
git checkout -b feature/my-changes  # create a branch
# make changes, commit, push
git checkout main  # switch to main
git pull
cd ..
git add frontend
git commit -m "Update frontend submodule"
```

### Common Issues & Solutions

#### Issue: "Submodule not initialized"
```bash
# Solution
git submodule update --init --recursive
```

#### Issue: "Submodule has uncommitted changes"
```bash
# Check what changed
cd frontend
git status

# Either commit them or reset
git add . && git commit -m "Changes"
# OR
git reset --hard
```

#### Issue: "Merge conflict in submodule"
```bash
# Update the submodule to the correct commit
cd frontend
git checkout <desired-commit>
cd ..
git add frontend
```

#### Issue: "Forgot to update parent repo after submodule changes"
```bash
# Parent repo will show modified submodule
git status  # shows: modified: frontend (new commits)

# Update parent to track new commit
git add frontend
git commit -m "Update frontend submodule"
```

---

## Docker Operations

### Build and Start All Services

```bash
docker-compose -f docker-compose-pipeline.yml up -d --build
```

### Watch Worker Logs

```bash
docker logs n8n-worker-1 -f
```

### Stop All Services

```bash
docker-compose -f docker-compose-pipeline.yml down
```

### Rebuild After Submodule Update

```bash
git pull
git submodule update --init --recursive
docker-compose -f docker-compose-pipeline.yml up -d --build
```

---

## Environment Configuration

1. Copy `.env.example` to `.env`
2. Update values for your environment
3. Never commit `.env` to git

---

## Deployment Checklist

- [ ] Pull latest parent repo: `git pull`
- [ ] Update submodules: `git submodule update --init --recursive`
- [ ] Verify submodule versions: `git submodule status`
- [ ] Update `.env` with production values
- [ ] Build and start: `docker-compose up -d --build`
- [ ] Check logs: `docker-compose logs -f`
- [ ] Verify services are healthy
- [ ] Tag deployment: `git tag prod-YYYY-MM-DD`
