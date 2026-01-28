# Program Versioning Guide

---
**Status**: 💡 **GUIDANCE** - Best practices for version management  
**Priority**: 🔴 **P0.9** - Must be implemented before first production deployment  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Proper version management is critical for tracking releases, communicating changes, managing deployments, and debugging production issues. This guide covers semantic versioning, version tracking, changelog management, and deployment strategies.

---

## Semantic Versioning (SemVer)

### The Standard: MAJOR.MINOR.PATCH

```
v2.4.7
│ │ │
│ │ └── PATCH: Bug fixes, no new features
│ └──── MINOR: New features, backward compatible
└────── MAJOR: Breaking changes, not backward compatible
```

### When to Increment

| Change Type | Increment | Example |
|------------|-----------|---------|
| **Breaking API change** | MAJOR | `1.5.3` → `2.0.0` |
| **Remove deprecated feature** | MAJOR | `3.2.1` → `4.0.0` |
| **Change database schema incompatibly** | MAJOR | `1.9.5` → `2.0.0` |
| **Add new feature** | MINOR | `1.2.3` → `1.3.0` |
| **Add endpoint** | MINOR | `2.1.0` → `2.2.0` |
| **Deprecate feature (still works)** | MINOR | `1.5.2` → `1.6.0` |
| **Bug fix** | PATCH | `1.2.3` → `1.2.4` |
| **Security patch** | PATCH | `3.1.0` → `3.1.1` |
| **Performance improvement** | PATCH | `2.5.1` → `2.5.2` |

### Pre-Release Versions

```
v1.0.0-alpha.1    # Alpha release (early testing)
v1.0.0-beta.2     # Beta release (feature complete, testing)
v1.0.0-rc.1       # Release candidate (ready for release)
v1.0.0            # Stable release
```

### Build Metadata

```
v1.2.3+20260127   # Build date
v1.2.3+abc123     # Git commit hash
v1.2.3+001        # Build number
```

---

## Implementation in Code

### package.json Version

```json
{
  "name": "myapp",
  "version": "1.2.3",
  "description": "My Application",
  "scripts": {
    "version": "npm run build && git add -A",
    "postversion": "git push && git push --tags"
  }
}
```

### Version Module

```typescript
// src/version.ts
import packageJson from '../package.json';

export interface VersionInfo {
  version: string;
  major: number;
  minor: number;
  patch: number;
  preRelease?: string;
  buildMetadata?: string;
  gitCommit?: string;
  buildDate?: string;
  nodeVersion: string;
}

export function getVersionInfo(): VersionInfo {
  const version = packageJson.version;
  const parts = version.split(/[\.\-\+]/);
  
  return {
    version,
    major: parseInt(parts[0] || '0'),
    minor: parseInt(parts[1] || '0'),
    patch: parseInt(parts[2] || '0'),
    preRelease: version.includes('-') ? version.split('-')[1]?.split('+')[0] : undefined,
    buildMetadata: version.includes('+') ? version.split('+')[1] : undefined,
    gitCommit: process.env.GIT_COMMIT || 'unknown',
    buildDate: process.env.BUILD_DATE || new Date().toISOString(),
    nodeVersion: process.version
  };
}

export const VERSION = getVersionInfo();
```

### Version Endpoint

```typescript
// src/routes/version.ts
import { Router } from 'express';
import { VERSION } from '../version';

const router = Router();

// Public version endpoint
router.get('/version', (req, res) => {
  res.json({
    version: VERSION.version,
    buildDate: VERSION.buildDate
  });
});

// Detailed version info (auth required)
router.get('/version/details', requireAuth, (req, res) => {
  res.json(VERSION);
});

export default router;
```

### Version in Logs

```typescript
// src/server.ts
import { VERSION } from './version';
import { logger } from './logger';

// Log version on startup
logger.info('Application starting', {
  version: VERSION.version,
  gitCommit: VERSION.gitCommit,
  buildDate: VERSION.buildDate,
  nodeVersion: VERSION.nodeVersion,
  environment: process.env.NODE_ENV
});
```

---

## Versioning Workflow

### Manual Versioning

```bash
# Bump version manually
npm version patch  # 1.2.3 -> 1.2.4
npm version minor  # 1.2.3 -> 1.3.0
npm version major  # 1.2.3 -> 2.0.0

# Bump pre-release
npm version prerelease --preid=alpha  # 1.2.3 -> 1.2.4-alpha.0
npm version prerelease --preid=beta   # 1.2.3 -> 1.2.4-beta.0
npm version prerelease --preid=rc     # 1.2.3 -> 1.2.4-rc.0

# What npm version does:
# 1. Updates version in package.json
# 2. Runs "version" script
# 3. Creates git commit "v1.2.4"
# 4. Creates git tag "v1.2.4"
# 5. Runs "postversion" script
```

### Automated Versioning with CI/CD

```bash
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Get all history for changelog
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Determine version bump
        id: version
        run: |
          # Check commit messages for version hints
          if git log -1 --pretty=%B | grep -q "BREAKING CHANGE"; then
            echo "bump=major" >> $GITHUB_OUTPUT
          elif git log -1 --pretty=%B | grep -q "^feat"; then
            echo "bump=minor" >> $GITHUB_OUTPUT
          else
            echo "bump=patch" >> $GITHUB_OUTPUT
          fi
      
      - name: Bump version
        run: npm version ${{ steps.version.outputs.bump }} --no-git-tag-version
      
      - name: Get new version
        id: package
        run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - name: Build
        run: npm run build
        env:
          GIT_COMMIT: ${{ github.sha }}
          BUILD_DATE: ${{ github.event.head_commit.timestamp }}
      
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ steps.package.outputs.version }}
          release_name: Release v${{ steps.package.outputs.version }}
          body: |
            ## Changes in v${{ steps.package.outputs.version }}
            
            See [CHANGELOG.md](CHANGELOG.md) for details.
```

### Conventional Commits for Auto-Versioning

```bash
# Commit message format
<type>(<scope>): <subject>

<body>

<footer>

# Examples:

# PATCH version (bug fix)
fix(api): resolve database connection timeout
fix(auth): correct session expiration logic

# MINOR version (new feature)
feat(api): add user export endpoint
feat(dashboard): add real-time metrics widget

# MAJOR version (breaking change)
feat(api)!: change authentication to OAuth2

BREAKING CHANGE: API now requires OAuth2 tokens instead of API keys

# Or
feat(api): migrate to GraphQL

BREAKING CHANGE: REST API endpoints removed, use GraphQL instead
```

---

## Changelog Management

### CHANGELOG.md Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]
### Added
- Feature in development, not yet released

## [1.2.3] - 2026-01-27

### Added
- New user export API endpoint
- Real-time metrics dashboard
- Support for bulk operations

### Changed
- Improved database query performance by 40%
- Updated authentication flow for better UX
- Migrated from REST to GraphQL for internal APIs

### Deprecated
- `/api/v1/users/export` endpoint (use `/api/v2/export/users` instead)

### Removed
- Legacy XML API support

### Fixed
- Fixed database connection timeout issues
- Resolved memory leak in background job processor
- Corrected timezone handling in reports

### Security
- Updated dependencies to patch CVE-2026-1234
- Implemented rate limiting on authentication endpoints

## [1.2.2] - 2026-01-20

### Fixed
- Critical bug in payment processing
- Session expiration not working correctly

### Security
- Patched SQL injection vulnerability in search endpoint

## [1.2.1] - 2026-01-15

### Changed
- Improved error messages for validation failures

### Fixed
- Fixed pagination issue in user list

## [1.2.0] - 2026-01-10

### Added
- User roles and permissions system
- Audit logging for admin actions
- Email notification system

### Changed
- Redesigned dashboard UI
- Updated database schema for better performance

## [1.1.0] - 2026-01-01

### Added
- Initial release
- User authentication
- Basic CRUD operations
- Admin dashboard

[Unreleased]: https://github.com/user/repo/compare/v1.2.3...HEAD
[1.2.3]: https://github.com/user/repo/compare/v1.2.2...v1.2.3
[1.2.2]: https://github.com/user/repo/compare/v1.2.1...v1.2.2
[1.2.1]: https://github.com/user/repo/compare/v1.2.0...v1.2.1
[1.2.0]: https://github.com/user/repo/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/user/repo/releases/tag/v1.1.0
```

### Auto-Generate Changelog

```bash
# Install standard-version
npm install --save-dev standard-version

# Add to package.json scripts
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  }
}

# Run release
npm run release

# What it does:
# 1. Bumps version in package.json
# 2. Updates CHANGELOG.md
# 3. Creates git commit and tag
```

---

## Version in Different Contexts

### Database Migrations

```typescript
// Track database schema version
export async function up(knex) {
  await knex.schema.createTable('schema_version', (table) => {
    table.increments('id').primary();
    table.string('version', 20).notNullable();
    table.timestamp('applied_at').defaultTo(knex.fn.now());
    table.text('description');
  });
  
  await knex('schema_version').insert({
    version: '1.0.0',
    description: 'Initial schema'
  });
}

// Check schema version compatibility
async function checkSchemaVersion() {
  const schemaVersion = await db('schema_version')
    .orderBy('applied_at', 'desc')
    .first();
  
  const requiredVersion = '1.2.0';
  
  if (semver.lt(schemaVersion.version, requiredVersion)) {
    throw new Error(
      `Database schema v${schemaVersion.version} is too old. ` +
      `Required: v${requiredVersion}. Please run migrations.`
    );
  }
}
```

### API Versioning

```typescript
// URL-based versioning
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// Header-based versioning
app.use((req, res, next) => {
  const apiVersion = req.headers['api-version'] || '1';
  req.apiVersion = apiVersion;
  next();
});

app.get('/api/users', (req, res) => {
  if (req.apiVersion === '2') {
    return res.json(getUsersV2());
  } else {
    return res.json(getUsersV1());
  }
});

// Deprecation warnings
app.use('/api/v1', (req, res, next) => {
  res.setHeader('X-API-Deprecation', 'API v1 is deprecated. Use v2.');
  res.setHeader('X-API-Sunset', '2026-12-31');
  next();
});
```

### Client Version Compatibility

```typescript
// Check client version compatibility
app.use((req, res, next) => {
  const clientVersion = req.headers['x-client-version'];
  
  if (!clientVersion) {
    return res.status(400).json({
      error: {
        message: 'Client version required',
        code: 'CLIENT_VERSION_REQUIRED'
      }
    });
  }
  
  const minVersion = '1.0.0';
  const maxVersion = '2.0.0';
  
  if (semver.lt(clientVersion, minVersion)) {
    return res.status(426).json({
      error: {
        message: 'Client version too old. Please update.',
        code: 'CLIENT_UPDATE_REQUIRED',
        minVersion,
        downloadUrl: 'https://myapp.com/download'
      }
    });
  }
  
  if (semver.gte(clientVersion, maxVersion)) {
    return res.status(426).json({
      error: {
        message: 'Client version too new. Server update required.',
        code: 'SERVER_UPDATE_REQUIRED'
      }
    });
  }
  
  next();
});
```

---

## Version Display in UI

### Frontend Version Display

```typescript
// React component
import React, { useEffect, useState } from 'react';

function VersionInfo() {
  const [version, setVersion] = useState(null);
  
  useEffect(() => {
    fetch('/api/version')
      .then(res => res.json())
      .then(data => setVersion(data));
  }, []);
  
  if (!version) return null;
  
  return (
    <div className="version-info">
      Version {version.version}
      {version.buildDate && (
        <span> • Built {new Date(version.buildDate).toLocaleDateString()}</span>
      )}
    </div>
  );
}

// Display in footer
export function Footer() {
  return (
    <footer>
      <VersionInfo />
    </footer>
  );
}
```

### Admin Panel Version

```typescript
// Admin version dashboard
app.get('/admin/version', requireAdmin, async (req, res) => {
  const [dbVersion] = await db('schema_version')
    .orderBy('applied_at', 'desc')
    .limit(1);
  
  const dependencies = {
    node: process.version,
    npm: process.env.npm_package_version,
    postgres: await db.raw('SELECT version()').then(r => r.rows[0].version),
    redis: await redis.info('server').then(info => 
      info.match(/redis_version:(.+)/)?.[1]
    )
  };
  
  res.json({
    application: VERSION,
    database: dbVersion,
    dependencies
  });
});
```

---

## Build-Time Version Injection

### Webpack/Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { execSync } from 'child_process';

const gitCommit = execSync('git rev-parse --short HEAD').toString().trim();
const buildDate = new Date().toISOString();

export default defineConfig({
  define: {
    '__APP_VERSION__': JSON.stringify(process.env.npm_package_version),
    '__GIT_COMMIT__': JSON.stringify(gitCommit),
    '__BUILD_DATE__': JSON.stringify(buildDate)
  }
});
```

### TypeScript Declarations

```typescript
// src/types/version.d.ts
declare const __APP_VERSION__: string;
declare const __GIT_COMMIT__: string;
declare const __BUILD_DATE__: string;
```

### Docker Build Args

```dockerfile
FROM node:18-alpine

ARG VERSION=unknown
ARG GIT_COMMIT=unknown
ARG BUILD_DATE=unknown

ENV APP_VERSION=$VERSION \
    GIT_COMMIT=$GIT_COMMIT \
    BUILD_DATE=$BUILD_DATE

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Version info file
RUN echo "{\"version\":\"$VERSION\",\"gitCommit\":\"$GIT_COMMIT\",\"buildDate\":\"$BUILD_DATE\"}" > version.json

CMD ["node", "dist/server.js"]
```

### Build Script

```bash
#!/bin/bash
# scripts/build-docker.sh

VERSION=$(node -p "require('./package.json').version")
GIT_COMMIT=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

docker build \
  --build-arg VERSION=$VERSION \
  --build-arg GIT_COMMIT=$GIT_COMMIT \
  --build-arg BUILD_DATE=$BUILD_DATE \
  -t myapp:$VERSION \
  -t myapp:latest \
  .

echo "Built myapp:$VERSION"
```

---

## Version Tracking in Production

### Startup Version Log

```typescript
// src/server.ts
import { VERSION } from './version';
import { logger } from './logger';

logger.info('='.repeat(60));
logger.info('Application Starting');
logger.info('='.repeat(60));
logger.info('Version Information', {
  version: VERSION.version,
  gitCommit: VERSION.gitCommit,
  buildDate: VERSION.buildDate,
  nodeVersion: VERSION.nodeVersion,
  environment: process.env.NODE_ENV,
  hostname: os.hostname(),
  platform: os.platform(),
  arch: os.arch()
});
logger.info('='.repeat(60));
```

### Error Reports with Version

```typescript
// Error tracking middleware
app.use((err, req, res, next) => {
  logger.error('Request error', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    version: VERSION.version,
    gitCommit: VERSION.gitCommit
  });
  
  // Send to error tracking service
  errorTracker.captureException(err, {
    tags: {
      version: VERSION.version,
      gitCommit: VERSION.gitCommit,
      environment: process.env.NODE_ENV
    }
  });
  
  res.status(500).json({
    error: {
      message: 'Internal server error',
      requestId: req.id,
      version: VERSION.version
    }
  });
});
```

### Deployment Tracking

```typescript
// Track deployments
async function recordDeployment() {
  await db('deployments').insert({
    version: VERSION.version,
    git_commit: VERSION.gitCommit,
    deployed_at: new Date(),
    deployed_by: process.env.DEPLOYED_BY || 'ci/cd',
    environment: process.env.NODE_ENV,
    hostname: os.hostname()
  });
  
  logger.info('Deployment recorded', {
    version: VERSION.version,
    environment: process.env.NODE_ENV
  });
}

// Call on startup
server.listen(PORT, async () => {
  await recordDeployment();
  logger.info(`Server started on port ${PORT}`);
});
```

---

## Version Comparison & Compatibility

### Semantic Version Utilities

```typescript
// utils/version.ts
import semver from 'semver';

export function isCompatible(
  clientVersion: string,
  serverVersion: string
): boolean {
  // Same major version = compatible
  return semver.major(clientVersion) === semver.major(serverVersion);
}

export function requiresUpdate(
  currentVersion: string,
  minimumVersion: string
): boolean {
  return semver.lt(currentVersion, minimumVersion);
}

export function isDeprecated(
  version: string,
  deprecatedVersion: string
): boolean {
  return semver.lt(version, deprecatedVersion);
}

// Usage
const clientVer = '1.5.2';
const serverVer = '1.8.0';
const minVer = '1.3.0';

if (!isCompatible(clientVer, serverVer)) {
  throw new Error('Client/server version mismatch');
}

if (requiresUpdate(clientVer, minVer)) {
  logger.warn('Client version outdated', { clientVer, minVer });
}
```

---

## Release Checklist

### Pre-Release Checklist

- [ ] All tests passing
- [ ] Code reviewed and merged to main
- [ ] CHANGELOG.md updated
- [ ] Version bumped appropriately
- [ ] Breaking changes documented
- [ ] Migration guide written (if breaking changes)
- [ ] API documentation updated
- [ ] Database migrations tested
- [ ] Backward compatibility verified

### Release Process

```bash
# 1. Ensure clean working directory
git status

# 2. Pull latest changes
git checkout main
git pull origin main

# 3. Run tests
npm test

# 4. Bump version and update changelog
npm run release  # or release:minor, release:major

# 5. Push changes and tags
git push --follow-tags origin main

# 6. Create GitHub release
gh release create v1.2.3 --generate-notes

# 7. Build and tag Docker image
./scripts/build-docker.sh

# 8. Push Docker image
docker push myapp:1.2.3
docker push myapp:latest

# 9. Deploy to staging
kubectl apply -f k8s/staging/

# 10. Verify staging deployment
curl https://staging-api.myapp.com/version

# 11. Deploy to production (after testing)
kubectl apply -f k8s/production/

# 12. Verify production deployment
curl https://api.myapp.com/version

# 13. Monitor for errors
# Check logs, metrics, error tracking

# 14. Announce release
# Post in Slack, send email, update status page
```

### Post-Release Checklist

- [ ] Staging deployment successful
- [ ] Production deployment successful
- [ ] Health checks passing
- [ ] No error rate increase
- [ ] Performance metrics normal
- [ ] User-facing announcement sent
- [ ] Internal team notified
- [ ] Documentation site updated
- [ ] Monitoring dashboards checked

---

## Version Strategy Best Practices

### DO ✅

1. **Use semantic versioning consistently**
   - Clear meaning for version changes
   - Predictable compatibility

2. **Tag every release in git**
   - Easy rollback
   - Clear history

3. **Maintain CHANGELOG.md**
   - Users know what changed
   - Debugging aid

4. **Include version in logs**
   - Essential for debugging
   - Track deployments

5. **Version your API**
   - Backward compatibility
   - Gradual migration

6. **Automate versioning**
   - Reduce human error
   - Consistent process

7. **Test before releasing**
   - Catch issues early
   - Maintain quality

### DON'T ❌

1. **Don't skip versions**
   - Creates confusion
   - Breaks tracking

2. **Don't reuse version numbers**
   - Even if you delete a release
   - Causes cache issues

3. **Don't make breaking changes in PATCH**
   - Violates semver contract
   - Breaks user trust

4. **Don't forget to tag releases**
   - Hard to find versions
   - Can't rollback easily

5. **Don't ignore API versioning**
   - Breaks client apps
   - Causes production issues

6. **Don't release without testing**
   - Introduces bugs
   - Damages reputation

7. **Don't forget changelogs**
   - Users don't know what changed
   - Support tickets increase

---

## Quick Reference

### Version Bump Commands

```bash
# Patch (1.2.3 -> 1.2.4)
npm version patch

# Minor (1.2.3 -> 1.3.0)
npm version minor

# Major (1.2.3 -> 2.0.0)
npm version major

# Pre-release
npm version prerelease --preid=beta  # 1.2.3 -> 1.2.4-beta.0
npm version prerelease                 # 1.2.4-beta.0 -> 1.2.4-beta.1
```

### Checking Versions

```bash
# Current version
node -p "require('./package.json').version"

# Git tag
git describe --tags

# Latest tag
git describe --tags --abbrev=0

# All versions
git tag -l
```

### Version Comparison

```bash
# In Node.js
const semver = require('semver');

semver.gt('1.2.3', '1.2.2')  // true
semver.lt('1.2.3', '1.3.0')  // true
semver.eq('1.2.3', '1.2.3')  // true
semver.major('1.2.3')        // 1
semver.minor('1.2.3')        // 2
semver.patch('1.2.3')        // 3
```

---

## Summary

### Key Principles

1. **Semantic Versioning**: MAJOR.MINOR.PATCH
2. **Tag Every Release**: Use git tags
3. **Maintain Changelog**: Document all changes
4. **Version Everything**: Code, API, database, client
5. **Include in Logs**: Track versions in production
6. **Automate**: Use tools to reduce errors

### Minimum Implementation

```typescript
// 1. Add version endpoint
app.get('/version', (req, res) => {
  res.json({
    version: process.env.npm_package_version,
    gitCommit: process.env.GIT_COMMIT,
    buildDate: process.env.BUILD_DATE
  });
});

// 2. Log version on startup
logger.info('Server starting', {
  version: process.env.npm_package_version
});

// 3. Follow semver for releases
// Use: npm version patch|minor|major
```

### Related Documentation
- [Deployment Runbook](./DEPLOYMENT.md)
- [API Documentation](./API.md)
- [Changelog](../CHANGELOG.md)

---

**Last Updated**: 2026-01-27  
**Owner**: Infrastructure Team  
**Review**: Quarterly or on major version changes