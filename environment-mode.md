# Environment Mode Management

---
**Status**: 💡 **GUIDANCE** - Best practices for environment management  
**Priority**: 🔴 **P0.6** - Critical before production launch  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Environment modes define how the application behaves in different contexts (development, staging, production). Proper environment management prevents production bugs, simplifies debugging, and ensures consistent deployments.

The Three Core Environments
Development (Local)
Purpose: Fast iteration, detailed debugging, developer productivity
Characteristics:

Verbose logging (DEBUG level)
Hot reload enabled
Pretty-printed logs (colors, human-readable)
Detailed error messages with stack traces
Mock external services
Relaxed security (for local testing)
Small dataset for fast iteration

Staging (Pre-Production)
Purpose: Production-like testing, QA validation, integration testing
Characteristics:

Production-like configuration
INFO level logging
JSON structured logs
Real external services (sandbox/test mode)
Production security settings
Realistic dataset (anonymized production data)
Performance monitoring enabled

Production (Live)
Purpose: Serve real users, maximum stability, security, performance
Characteristics:

Minimal logging (INFO/WARN level)
JSON structured logs
All security hardened
Real external services
Monitoring and alerting enabled
Optimized for performance
Graceful degradation


Environment Detection
Using NODE_ENV
typescript// config/environment.ts

export type Environment = 'development' | 'staging' | 'production' | 'test';

export const environment: Environment = 
  (process.env.NODE_ENV as Environment) || 'development';

export const isDevelopment = environment === 'development';
export const isStaging = environment === 'staging';
export const isProduction = environment === 'production';
export const isTest = environment === 'test';

// Helper functions
export function requireProduction(message?: string): void {
  if (!isProduction) {
    throw new Error(message || 'This operation is only allowed in production');
  }
}

export function requireNonProduction(message?: string): void {
  if (isProduction) {
    throw new Error(message || 'This operation is not allowed in production');
  }
}

// Usage examples
if (isDevelopment) {
  console.log('Development mode - enabling hot reload');
}

if (isProduction) {
  // Enable production optimizations
  app.set('trust proxy', 1);
}

// Prevent dangerous operations in production
app.delete('/api/admin/reset-database', (req, res) => {
  requireNonProduction('Cannot reset database in production');
  // ... reset logic
});

Environment-Specific Configuration
Configuration File Structure
config/
├── index.ts              # Main config export
├── environment.ts        # Environment detection
├── development.ts        # Development config
├── staging.ts           # Staging config
├── production.ts        # Production config
└── test.ts              # Test config
Base Configuration Pattern
typescript// config/index.ts
import { environment } from './environment';
import developmentConfig from './development';
import stagingConfig from './staging';
import productionConfig from './production';
import testConfig from './test';

interface Config {
  // App settings
  port: number;
  appName: string;
  apiVersion: string;
  
  // Database
  database: {
    url: string;
    pool: {
      min: number;
      max: number;
    };
    ssl: boolean;
    logging: boolean;
  };
  
  // Redis
  redis: {
    url: string;
    tls: boolean;
  };
  
  // Logging
  logging: {
    level: 'trace' | 'debug' | 'info' | 'warn' | 'error';
    pretty: boolean;
    destination?: string;
  };
  
  // Security
  security: {
    cors: {
      origins: string[];
      credentials: boolean;
    };
    rateLimit: {
      windowMs: number;
      max: number;
    };
    session: {
      secret: string;
      secure: boolean;
    };
  };
  
  // External services
  services: {
    stripe: {
      apiKey: string;
      webhookSecret: string;
      testMode: boolean;
    };
    sendgrid: {
      apiKey: string;
      fromEmail: string;
    };
  };
  
  // Features flags
  features: {
    enableCache: boolean;
    enableBackgroundJobs: boolean;
    enableMetrics: boolean;
    enableDebugEndpoints: boolean;
  };
}

const configs: Record<string, Config> = {
  development: developmentConfig,
  staging: stagingConfig,
  production: productionConfig,
  test: testConfig
};

export const config = configs[environment];

// Validate config on startup
if (!config) {
  throw new Error(`Invalid environment: ${environment}`);
}

export default config;
Development Configuration
typescript// config/development.ts
import { Config } from './index';

const developmentConfig: Config = {
  port: 3000,
  appName: 'MyApp Development',
  apiVersion: 'v1',
  
  database: {
    url: process.env.DATABASE_URL || 'postgresql://localhost/myapp_dev',
    pool: {
      min: 2,
      max: 5 // Lower pool for dev
    },
    ssl: false,
    logging: true // Log all SQL queries
  },
  
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    tls: false
  },
  
  logging: {
    level: 'debug',
    pretty: true, // Human-readable logs
    destination: undefined // Log to console
  },
  
  security: {
    cors: {
      origins: ['http://localhost:3000', 'http://localhost:3001'],
      credentials: true
    },
    rateLimit: {
      windowMs: 15 * 60 * 1000,
      max: 1000 // Very generous for dev
    },
    session: {
      secret: 'dev-secret-change-in-production',
      secure: false // Allow HTTP in dev
    }
  },
  
  services: {
    stripe: {
      apiKey: process.env.STRIPE_TEST_KEY || 'sk_test_...',
      webhookSecret: 'whsec_test_...',
      testMode: true
    },
    sendgrid: {
      apiKey: process.env.SENDGRID_API_KEY || 'fake-key-for-dev',
      fromEmail: 'dev@localhost'
    }
  },
  
  features: {
    enableCache: false, // Don't cache in dev
    enableBackgroundJobs: true,
    enableMetrics: false,
    enableDebugEndpoints: true // Enable /debug routes
  }
};

export default developmentConfig;
Staging Configuration
typescript// config/staging.ts
import { Config } from './index';

const stagingConfig: Config = {
  port: parseInt(process.env.PORT || '3000'),
  appName: 'MyApp Staging',
  apiVersion: 'v1',
  
  database: {
    url: process.env.DATABASE_URL!, // Required in staging
    pool: {
      min: 5,
      max: 20 // Production-like
    },
    ssl: true, // Always use SSL
    logging: false // Don't log queries
  },
  
  redis: {
    url: process.env.REDIS_URL!,
    tls: true
  },
  
  logging: {
    level: 'info',
    pretty: false, // JSON logs
    destination: '/var/log/app/app.log'
  },
  
  security: {
    cors: {
      origins: [
        'https://staging.myapp.com',
        'https://staging-admin.myapp.com'
      ],
      credentials: true
    },
    rateLimit: {
      windowMs: 15 * 60 * 1000,
      max: 100 // Production-like limits
    },
    session: {
      secret: process.env.SESSION_SECRET!,
      secure: true // HTTPS only
    }
  },
  
  services: {
    stripe: {
      apiKey: process.env.STRIPE_TEST_KEY!,
      webhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
      testMode: true // Still test mode
    },
    sendgrid: {
      apiKey: process.env.SENDGRID_API_KEY!,
      fromEmail: 'staging@myapp.com'
    }
  },
  
  features: {
    enableCache: true,
    enableBackgroundJobs: true,
    enableMetrics: true,
    enableDebugEndpoints: true // Keep for staging testing
  }
};

export default stagingConfig;
Production Configuration
typescript// config/production.ts
import { Config } from './index';

const productionConfig: Config = {
  port: parseInt(process.env.PORT || '3000'),
  appName: 'MyApp',
  apiVersion: 'v1',
  
  database: {
    url: process.env.DATABASE_URL!,
    pool: {
      min: 10,
      max: 50 // Scale for production
    },
    ssl: true,
    logging: false
  },
  
  redis: {
    url: process.env.REDIS_URL!,
    tls: true
  },
  
  logging: {
    level: 'info',
    pretty: false,
    destination: '/var/log/app/app.log'
  },
  
  security: {
    cors: {
      origins: [
        'https://myapp.com',
        'https://www.myapp.com',
        'https://admin.myapp.com'
      ],
      credentials: true
    },
    rateLimit: {
      windowMs: 15 * 60 * 1000,
      max: 100
    },
    session: {
      secret: process.env.SESSION_SECRET!,
      secure: true
    }
  },
  
  services: {
    stripe: {
      apiKey: process.env.STRIPE_LIVE_KEY!,
      webhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
      testMode: false // LIVE mode
    },
    sendgrid: {
      apiKey: process.env.SENDGRID_API_KEY!,
      fromEmail: 'noreply@myapp.com'
    }
  },
  
  features: {
    enableCache: true,
    enableBackgroundJobs: true,
    enableMetrics: true,
    enableDebugEndpoints: false // DISABLED in production
  }
};

export default productionConfig;

Environment Variable Management
.env File Structure
bash# .env.development (committed to git)
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://localhost/myapp_dev
REDIS_URL=redis://localhost:6379
LOG_LEVEL=debug

# Development API keys (test mode)
STRIPE_TEST_KEY=sk_test_xxxxx
SENDGRID_API_KEY=not-needed-in-dev

# .env.staging (committed to git, uses placeholders)
NODE_ENV=staging
PORT=3000
DATABASE_URL=${DATABASE_URL}
REDIS_URL=${REDIS_URL}
SESSION_SECRET=${SESSION_SECRET}
STRIPE_TEST_KEY=${STRIPE_TEST_KEY}
STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
SENDGRID_API_KEY=${SENDGRID_API_KEY}

# .env.production (committed to git, uses placeholders)
NODE_ENV=production
PORT=3000
DATABASE_URL=${DATABASE_URL}
REDIS_URL=${REDIS_URL}
SESSION_SECRET=${SESSION_SECRET}
STRIPE_LIVE_KEY=${STRIPE_LIVE_KEY}
STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
SENDGRID_API_KEY=${SENDGRID_API_KEY}

# .env.local (NEVER committed, developer-specific overrides)
DATABASE_URL=postgresql://localhost/myapp_custom
STRIPE_TEST_KEY=my_personal_test_key
Environment Loading Priority
typescript// Load environment variables with priority
import dotenv from 'dotenv';
import path from 'path';

// 1. Load base .env
dotenv.config({ path: path.join(__dirname, '../.env') });

// 2. Load environment-specific .env (overrides base)
dotenv.config({ 
  path: path.join(__dirname, `../.env.${process.env.NODE_ENV}`) 
});

// 3. Load local overrides (overrides everything)
dotenv.config({ 
  path: path.join(__dirname, '../.env.local'),
  override: true 
});

// Priority order:
// .env.local > .env.{NODE_ENV} > .env > system env vars

Environment Validation
Using Zod for Validation
typescript// config/validation.ts
import { z } from 'zod';

const envSchema = z.object({
  // Node
  NODE_ENV: z.enum(['development', 'staging', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  
  // Database
  DATABASE_URL: z.string().url(),
  
  // Redis
  REDIS_URL: z.string().url(),
  
  // Security
  SESSION_SECRET: z.string().min(32).optional().refine(
    (val) => {
      if (process.env.NODE_ENV === 'production' && !val) {
        return false;
      }
      return true;
    },
    { message: 'SESSION_SECRET is required in production' }
  ),
  
  // External services
  STRIPE_TEST_KEY: z.string().optional(),
  STRIPE_LIVE_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  SENDGRID_API_KEY: z.string().optional(),
  
  // Logging
  LOG_LEVEL: z.enum(['trace', 'debug', 'info', 'warn', 'error']).default('info')
}).refine(
  (data) => {
    // In production, require live Stripe key
    if (data.NODE_ENV === 'production') {
      return !!data.STRIPE_LIVE_KEY && !!data.STRIPE_WEBHOOK_SECRET;
    }
    return true;
  },
  { message: 'Stripe live keys required in production' }
);

// Validate and export
export const env = envSchema.parse(process.env);

// Now TypeScript knows these are validated!
// env.DATABASE_URL is definitely a string
// env.PORT is definitely a number
Validation on Startup
typescript// server.ts
import { env } from './config/validation';
import { logger } from './logger';

// Validate environment before starting server
try {
  logger.info('Environment validated successfully', {
    nodeEnv: env.NODE_ENV,
    port: env.PORT,
    // Don't log secrets!
  });
} catch (error) {
  logger.error('Environment validation failed', { error });
  process.exit(1);
}

// Start server
server.listen(env.PORT, () => {
  logger.info(`Server started in ${env.NODE_ENV} mode on port ${env.PORT}`);
});

Environment-Specific Behavior
Conditional Features
typescript// middleware/debug.ts
import { Router } from 'express';
import { isDevelopment, isStaging } from '../config/environment';

const debugRouter = Router();

// Only enable debug endpoints in non-production
if (isDevelopment || isStaging) {
  debugRouter.get('/debug/config', (req, res) => {
    res.json({
      environment: process.env.NODE_ENV,
      config: {
        // Safe config to expose (no secrets!)
        port: config.port,
        database: {
          poolSize: config.database.pool.max
        }
      }
    });
  });
  
  debugRouter.get('/debug/routes', (req, res) => {
    // List all registered routes
    const routes = [];
    app._router.stack.forEach((middleware) => {
      if (middleware.route) {
        routes.push({
          path: middleware.route.path,
          methods: Object.keys(middleware.route.methods)
        });
      }
    });
    res.json({ routes });
  });
  
  debugRouter.post('/debug/seed-database', async (req, res) => {
    requireNonProduction('Cannot seed in production');
    await seedDatabase();
    res.json({ success: true });
  });
}

export default debugRouter;
Environment-Specific Middleware
typescript// server.ts
import { isDevelopment, isProduction } from './config/environment';
import morgan from 'morgan';

// Request logging
if (isDevelopment) {
  // Detailed logging in dev
  app.use(morgan('dev'));
} else {
  // Minimal logging in production
  app.use(morgan('combined'));
}

// CORS
if (isDevelopment) {
  // Allow all origins in dev
  app.use(cors({ origin: '*' }));
} else {
  // Strict CORS in production
  app.use(cors({ origin: config.security.cors.origins }));
}

// Error handling
app.use((err, req, res, next) => {
  logger.error('Request error', { error: err, url: req.url });
  
  res.status(err.status || 500).json({
    error: {
      message: err.message,
      // Only show stack traces in development
      ...(isDevelopment && { stack: err.stack })
    }
  });
});

Feature Flags
Simple Feature Flag System
typescript// config/features.ts
import { environment } from './environment';

interface FeatureFlags {
  enableNewDashboard: boolean;
  enableBetaFeatures: boolean;
  enableExperimentalAPI: boolean;
  enableAdvancedAnalytics: boolean;
}

const featureFlags: Record<string, FeatureFlags> = {
  development: {
    enableNewDashboard: true,
    enableBetaFeatures: true,
    enableExperimentalAPI: true,
    enableAdvancedAnalytics: true
  },
  staging: {
    enableNewDashboard: true,
    enableBetaFeatures: true,
    enableExperimentalAPI: false,
    enableAdvancedAnalytics: true
  },
  production: {
    enableNewDashboard: false, // Gradual rollout
    enableBetaFeatures: false,
    enableExperimentalAPI: false,
    enableAdvancedAnalytics: true
  }
};

export const features = featureFlags[environment];

// Usage
import { features } from './config/features';

app.get('/api/dashboard', (req, res) => {
  if (features.enableNewDashboard) {
    return res.json(getNewDashboard());
  } else {
    return res.json(getOldDashboard());
  }
});
Advanced Feature Flags (LaunchDarkly-style)
typescript// config/feature-flags.ts
class FeatureFlagManager {
  private flags: Map<string, boolean> = new Map();
  
  constructor() {
    this.loadFlags();
  }
  
  private loadFlags() {
    // Load from database, config service, or environment
    const flagsFromEnv = process.env.FEATURE_FLAGS?.split(',') || [];
    flagsFromEnv.forEach(flag => this.flags.set(flag, true));
  }
  
  isEnabled(flagName: string, defaultValue = false): boolean {
    return this.flags.get(flagName) ?? defaultValue;
  }
  
  enable(flagName: string) {
    this.flags.set(flagName, true);
  }
  
  disable(flagName: string) {
    this.flags.set(flagName, false);
  }
  
  // Percentage rollout
  isEnabledForUser(flagName: string, userId: string, percentage: number): boolean {
    if (!this.isEnabled(flagName)) return false;
    
    // Hash user ID to determine if they're in the rollout percentage
    const hash = this.hashUserId(userId);
    return (hash % 100) < percentage;
  }
  
  private hashUserId(userId: string): number {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }
}

export const featureFlags = new FeatureFlagManager();

// Usage
app.get('/api/new-feature', (req, res) => {
  const userId = req.user.id;
  
  // 20% rollout
  if (featureFlags.isEnabledForUser('new-algorithm', userId, 20)) {
    return res.json(getNewAlgorithmResults());
  } else {
    return res.json(getOldAlgorithmResults());
  }
});

Environment Detection Helpers
Utility Functions
typescript// utils/environment.ts
import { environment } from '../config/environment';

export function getEnvironmentColor(): string {
  switch (environment) {
    case 'development': return 'green';
    case 'staging': return 'yellow';
    case 'production': return 'red';
    default: return 'gray';
  }
}

export function getEnvironmentEmoji(): string {
  switch (environment) {
    case 'development': return '🛠️';
    case 'staging': return '🧪';
    case 'production': return '🚀';
    default: return '❓';
  }
}

export function shouldEnableDebugMode(): boolean {
  return environment !== 'production';
}

export function getLogRetentionDays(): number {
  switch (environment) {
    case 'development': return 7;
    case 'staging': return 30;
    case 'production': return 90;
    default: return 7;
  }
}

export function getDatabasePoolSize(): number {
  switch (environment) {
    case 'development': return 5;
    case 'staging': return 20;
    case 'production': return 50;
    default: return 5;
  }
}

Testing Environments
Test Configuration
typescript// config/test.ts
import { Config } from './index';

const testConfig: Config = {
  port: 3001, // Different port
  appName: 'MyApp Test',
  apiVersion: 'v1',
  
  database: {
    // Use separate test database
    url: process.env.DATABASE_URL || 'postgresql://localhost/myapp_test',
    pool: {
      min: 1,
      max: 3 // Small pool for tests
    },
    ssl: false,
    logging: false // Silent during tests
  },
  
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379/1', // Different DB
    tls: false
  },
  
  logging: {
    level: 'error', // Only log errors during tests
    pretty: false,
    destination: undefined
  },
  
  security: {
    cors: {
      origins: ['*'],
      credentials: true
    },
    rateLimit: {
      windowMs: 15 * 60 * 1000,
      max: 999999 // No rate limiting in tests
    },
    session: {
      secret: 'test-secret',
      secure: false
    }
  },
  
  services: {
    stripe: {
      apiKey: 'sk_test_mock',
      webhookSecret: 'whsec_test_mock',
      testMode: true
    },
    sendgrid: {
      apiKey: 'mock-key',
      fromEmail: 'test@test.com'
    }
  },
  
  features: {
    enableCache: false, // Disable caching in tests
    enableBackgroundJobs: false, // Disable async jobs
    enableMetrics: false,
    enableDebugEndpoints: true
  }
};

export default testConfig;
Test Setup/Teardown
typescript// tests/setup.ts
import { beforeAll, afterAll, beforeEach } from '@jest/globals';
import { db } from '../src/database';

beforeAll(async () => {
  // Ensure we're in test environment
  if (process.env.NODE_ENV !== 'test') {
    throw new Error('Tests must run in test environment');
  }
  
  // Run migrations
  await db.migrate.latest();
});

beforeEach(async () => {
  // Clean database before each test
  await db('users').del();
  await db('records').del();
});

afterAll(async () => {
  // Cleanup
  await db.destroy();
});

Docker & Container Environments
Dockerfile with Environment Support
dockerfileFROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Build TypeScript
RUN npm run build

# Default to production
ENV NODE_ENV=production

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Start app
CMD ["node", "dist/server.js"]
Docker Compose with Multiple Environments
yaml# docker-compose.yml (development)
version: '3.8'

services:
  app:
    build: .
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp_dev
      - REDIS_URL=redis://redis:6379
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src
      - ./logs:/app/logs
    depends_on:
      - db
      - redis
    command: npm run dev

  db:
    image: postgres:14
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp_dev
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

# docker-compose.staging.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - NODE_ENV=staging
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SESSION_SECRET=${SESSION_SECRET}
    ports:
      - "3000:3000"
    restart: unless-stopped

Kubernetes Environment Management
ConfigMaps per Environment
yaml# k8s/configmap-development.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  NODE_ENV: "development"
  LOG_LEVEL: "debug"
  ENABLE_DEBUG_ENDPOINTS: "true"

---
# k8s/configmap-production.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  ENABLE_DEBUG_ENDPOINTS: "false"
Deployment with Environment
yamlapiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: myapp:latest
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV

Environment Checklist
Pre-Deployment Checklist
Development:

 .env.development file exists
 Local database accessible
 Local Redis accessible
 Hot reload working
 Debug endpoints enabled
 Detailed logging enabled

Staging:

 .env.staging configured
 Staging database accessible
 Staging Redis accessible
 SSL/TLS enabled
 Production-like configuration
 Monitoring enabled
 Test data seeded

Production:

 .env.production configured
 All secrets in secrets manager
 Database connections validated
 Redis connections validated
 SSL/TLS enforced
 Debug endpoints disabled
 Rate limiting enabled
 Monitoring and alerting enabled
 Backups configured
 Health checks passing


Common Pitfalls
Problem 1: Secrets in Git
❌ Wrong:
bash# .env (committed to git)
DATABASE_URL=postgresql://user:password@prod-db.com/db
SESSION_SECRET=my-secret-key
✅ Right:
bash# .env.example (committed to git)
DATABASE_URL=postgresql://localhost/myapp_dev
SESSION_SECRET=change-me-in-production

# .env (NOT committed, in .gitignore)
DATABASE_URL=postgresql://real-password@prod-db.com/db
SESSION_SECRET=actual-secure-secret
Problem 2: No Environment Validation
❌ Wrong:
typescriptconst dbUrl = process.env.DATABASE_URL; // Could be undefined!
db.connect(dbUrl); // Crashes at runtime
✅ Right:
typescriptimport { env } from './config/validation';
db.connect(env.DATABASE_URL); // TypeScript knows it's a string
Problem 3: Production Secrets in Code
❌ Wrong:
typescriptconst stripeKey = isProduction 
  ? 'sk_live_hardcoded_key' 
  : 'sk_test_key';
✅ Right:
typescriptconst stripeKey = env.STRIPE_KEY; // From environment
Problem 4: Same Config Everywhere
❌ Wrong:
typescriptconst poolSize = 50; // Always 50, even in dev
✅ Right:
typescriptconst poolSize = config.database.pool.max; // Environment-specific

Summary
Key Principles

Never hardcode environment-specific values
Validate environment on startup
Use different configs per environment
Keep secrets out of git
Make it impossible to use production secrets in dev
Fail fast on missing config

Quick Start
bash# 1. Copy example env
cp .env.example .env.local

# 2. Fill in values
# Edit .env.local

# 3. Start in development
NODE_ENV=development npm run dev

# 4. Test staging config
NODE_ENV=staging npm start

# 5. Deploy to production
NODE_ENV=production npm start