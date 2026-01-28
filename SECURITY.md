# Security Documentation
## Authentication, Authorization, Rate Limiting & Security Best Practices

---
**Status**: 💡 **GUIDANCE** - Security best practices and implementation patterns  
**Priority**: 🔴 **P0** - Critical before production launch  
**Last Updated**: 2026-01-28  
**Owned By**: Security Team, Backend Team  

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Authorization](#authorization)
4. [Rate Limiting](#rate-limiting)
5. [Input Validation](#input-validation)
6. [Secrets Management](#secrets-management)
7. [HTTPS & TLS](#https--tls)
8. [Security Headers](#security-headers)
9. [CORS Configuration](#cors-configuration)
10. [Dependency Security](#dependency-security)
11. [Security Monitoring](#security-monitoring)
12. [Incident Response](#incident-response)

---

## Overview

### Security Principles

1. **Defense in Depth**: Multiple layers of security
2. **Least Privilege**: Minimum permissions necessary
3. **Fail Secure**: Errors should deny access, not grant it
4. **Secure by Default**: Security enabled out of the box
5. **Zero Trust**: Never trust, always verify

### Threat Model

```
┌─────────────────────────────────────────┐
│  External Threats                       │
│  - DDoS attacks                        │
│  - Brute force attempts                │
│  - SQL injection                       │
│  - XSS attacks                         │
│  - CSRF attacks                        │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Defenses                              │
│  - Rate limiting                       │
│  - Input validation                    │
│  - Parameterized queries               │
│  - CSP headers                         │
│  - CSRF tokens                         │
└─────────────────────────────────────────┘
```

---

## Authentication

### JWT Authentication

```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import { randomBytes } from 'crypto';

interface TokenPayload {
  userId: string;
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

export class AuthService {
  private readonly JWT_SECRET: string;
  private readonly JWT_REFRESH_SECRET: string;
  private readonly ACCESS_TOKEN_EXPIRY = '15m';
  private readonly REFRESH_TOKEN_EXPIRY = '7d';
  private readonly SALT_ROUNDS = 12;
  
  constructor() {
    this.JWT_SECRET = process.env.JWT_SECRET!;
    this.JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;
    
    if (!this.JWT_SECRET || !this.JWT_REFRESH_SECRET) {
      throw new Error('JWT secrets not configured');
    }
  }
  
  // Register new user
  async register(email: string, password: string): Promise<User> {
    // Validate password strength
    this.validatePasswordStrength(password);
    
    // Hash password
    const passwordHash = await bcrypt.hash(password, this.SALT_ROUNDS);
    
    // Create user
    const user = await db.users.create({
      email,
      passwordHash,
      emailVerified: false,
      createdAt: new Date(),
    });
    
    // Send verification email
    await this.sendVerificationEmail(user);
    
    return user;
  }
  
  // Login user
  async login(email: string, password: string): Promise<{
    accessToken: string;
    refreshToken: string;
    user: User;
  }> {
    // Find user
    const user = await db.users.findByEmail(email);
    if (!user) {
      // Don't reveal if user exists
      throw new Error('Invalid credentials');
    }
    
    // Check if account is locked
    if (user.lockedUntil && user.lockedUntil > new Date()) {
      throw new Error('Account temporarily locked. Try again later.');
    }
    
    // Verify password
    const isValid = await bcrypt.compare(password, user.passwordHash);
    
    if (!isValid) {
      // Track failed attempts
      await this.handleFailedLogin(user.id);
      throw new Error('Invalid credentials');
    }
    
    // Reset failed attempts on successful login
    await db.users.update(user.id, {
      failedLoginAttempts: 0,
      lastLoginAt: new Date(),
    });
    
    // Generate tokens
    const accessToken = this.generateAccessToken(user);
    const refreshToken = await this.generateRefreshToken(user);
    
    return { accessToken, refreshToken, user };
  }
  
  // Generate access token
  private generateAccessToken(user: User): string {
    const payload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    };
    
    return jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: this.ACCESS_TOKEN_EXPIRY,
      issuer: 'myapp-api',
      audience: 'myapp-client',
    });
  }
  
  // Generate refresh token
  private async generateRefreshToken(user: User): Promise<string> {
    const token = randomBytes(40).toString('hex');
    const expiresAt = new Date();
    expiresAt.setDate(expiresAt.getDate() + 7); // 7 days
    
    // Store in database
    await db.refreshTokens.create({
      token,
      userId: user.id,
      expiresAt,
    });
    
    return token;
  }
  
  // Verify access token
  verifyAccessToken(token: string): TokenPayload {
    try {
      return jwt.verify(token, this.JWT_SECRET, {
        issuer: 'myapp-api',
        audience: 'myapp-client',
      }) as TokenPayload;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('Token expired');
      }
      throw new Error('Invalid token');
    }
  }
  
  // Refresh access token
  async refreshAccessToken(refreshToken: string): Promise<string> {
    // Find refresh token
    const tokenRecord = await db.refreshTokens.findByToken(refreshToken);
    
    if (!tokenRecord) {
      throw new Error('Invalid refresh token');
    }
    
    // Check if expired
    if (tokenRecord.expiresAt < new Date()) {
      await db.refreshTokens.delete(tokenRecord.id);
      throw new Error('Refresh token expired');
    }
    
    // Get user
    const user = await db.users.findById(tokenRecord.userId);
    if (!user) {
      throw new Error('User not found');
    }
    
    // Generate new access token
    return this.generateAccessToken(user);
  }
  
  // Handle failed login
  private async handleFailedLogin(userId: string): Promise<void> {
    const user = await db.users.findById(userId);
    if (!user) return;
    
    const attempts = (user.failedLoginAttempts || 0) + 1;
    
    // Lock account after 5 failed attempts
    if (attempts >= 5) {
      const lockUntil = new Date();
      lockUntil.setMinutes(lockUntil.getMinutes() + 15); // 15 minutes
      
      await db.users.update(userId, {
        failedLoginAttempts: attempts,
        lockedUntil: lockUntil,
      });
      
      logger.warn({ userId, attempts }, 'Account locked due to failed login attempts');
    } else {
      await db.users.update(userId, {
        failedLoginAttempts: attempts,
      });
    }
  }
  
  // Validate password strength
  private validatePasswordStrength(password: string): void {
    if (password.length < 12) {
      throw new Error('Password must be at least 12 characters');
    }
    
    const hasUpper = /[A-Z]/.test(password);
    const hasLower = /[a-z]/.test(password);
    const hasNumber = /[0-9]/.test(password);
    const hasSpecial = /[^A-Za-z0-9]/.test(password);
    
    if (!hasUpper || !hasLower || !hasNumber || !hasSpecial) {
      throw new Error('Password must contain uppercase, lowercase, number, and special character');
    }
  }
  
  // Send verification email
  private async sendVerificationEmail(user: User): Promise<void> {
    const token = randomBytes(32).toString('hex');
    
    await db.verificationTokens.create({
      token,
      userId: user.id,
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
    });
    
    // Send email (implement email service)
    await emailService.send({
      to: user.email,
      subject: 'Verify your email',
      template: 'verify-email',
      data: { token },
    });
  }
}

// Express middleware
export function authenticateJWT(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const payload = authService.verifyAccessToken(token);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### API Key Authentication

```typescript
import { createHash } from 'crypto';

export class APIKeyService {
  // Generate API key
  async createAPIKey(userId: string, name: string): Promise<{
    key: string;
    keyId: string;
  }> {
    // Generate random key
    const key = `sk_${randomBytes(32).toString('hex')}`;
    
    // Hash for storage (never store plain key)
    const keyHash = createHash('sha256').update(key).digest('hex');
    
    // Store in database
    const apiKey = await db.apiKeys.create({
      userId,
      name,
      keyHash,
      createdAt: new Date(),
      lastUsedAt: null,
    });
    
    // Return key ONCE (user must save it)
    return {
      key,
      keyId: apiKey.id,
    };
  }
  
  // Validate API key
  async validateAPIKey(key: string): Promise<User | null> {
    const keyHash = createHash('sha256').update(key).digest('hex');
    
    const apiKey = await db.apiKeys.findByHash(keyHash);
    if (!apiKey) return null;
    
    // Update last used
    await db.apiKeys.update(apiKey.id, {
      lastUsedAt: new Date(),
    });
    
    // Get user
    return await db.users.findById(apiKey.userId);
  }
}

// Middleware
export function authenticateAPIKey(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const apiKey = req.headers['x-api-key'] as string;
  
  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }
  
  const user = await apiKeyService.validateAPIKey(apiKey);
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid API key' });
  }
  
  req.user = user;
  next();
}
```

---

## Authorization

### Role-Based Access Control (RBAC)

```typescript
enum Role {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
  GUEST = 'guest',
}

enum Permission {
  READ_USERS = 'read:users',
  WRITE_USERS = 'write:users',
  DELETE_USERS = 'delete:users',
  READ_POSTS = 'read:posts',
  WRITE_POSTS = 'write:posts',
  DELETE_POSTS = 'delete:posts',
  MODERATE_CONTENT = 'moderate:content',
}

const rolePermissions: Record<Role, Permission[]> = {
  [Role.ADMIN]: [
    Permission.READ_USERS,
    Permission.WRITE_USERS,
    Permission.DELETE_USERS,
    Permission.READ_POSTS,
    Permission.WRITE_POSTS,
    Permission.DELETE_POSTS,
    Permission.MODERATE_CONTENT,
  ],
  [Role.MODERATOR]: [
    Permission.READ_USERS,
    Permission.READ_POSTS,
    Permission.MODERATE_CONTENT,
  ],
  [Role.USER]: [
    Permission.READ_POSTS,
    Permission.WRITE_POSTS,
  ],
  [Role.GUEST]: [
    Permission.READ_POSTS,
  ],
};

export class AuthorizationService {
  hasPermission(user: User, permission: Permission): boolean {
    const permissions = rolePermissions[user.role as Role] || [];
    return permissions.includes(permission);
  }
  
  hasAnyPermission(user: User, permissions: Permission[]): boolean {
    return permissions.some(p => this.hasPermission(user, p));
  }
  
  hasAllPermissions(user: User, permissions: Permission[]): boolean {
    return permissions.every(p => this.hasPermission(user, p));
  }
}

// Middleware
export function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    
    if (!authorizationService.hasPermission(req.user, permission)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    next();
  };
}

export function requireRole(role: Role) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    
    if (req.user.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    next();
  };
}

// Usage
app.get('/admin/users',
  authenticateJWT,
  requirePermission(Permission.READ_USERS),
  async (req, res) => {
    const users = await db.users.findAll();
    res.json(users);
  }
);

app.delete('/admin/users/:id',
  authenticateJWT,
  requireRole(Role.ADMIN),
  async (req, res) => {
    await db.users.delete(req.params.id);
    res.json({ success: true });
  }
);
```

### Resource-Based Authorization

```typescript
export class ResourceAuthorization {
  // Check if user can access resource
  async canAccessPost(user: User, postId: string): Promise<boolean> {
    const post = await db.posts.findById(postId);
    if (!post) return false;
    
    // Admins can access everything
    if (user.role === Role.ADMIN) return true;
    
    // Users can access their own posts
    if (post.authorId === user.id) return true;
    
    // Users can access public posts
    if (post.visibility === 'public') return true;
    
    return false;
  }
  
  // Check if user can modify resource
  async canModifyPost(user: User, postId: string): Promise<boolean> {
    const post = await db.posts.findById(postId);
    if (!post) return false;
    
    // Admins can modify everything
    if (user.role === Role.ADMIN) return true;
    
    // Users can only modify their own posts
    return post.authorId === user.id;
  }
}

// Middleware
export function requireResourceAccess(
  resourceType: 'post' | 'user',
  resourceIdParam: string = 'id'
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const resourceId = req.params[resourceIdParam];
    
    let canAccess = false;
    
    if (resourceType === 'post') {
      canAccess = await resourceAuth.canAccessPost(req.user, resourceId);
    }
    // Add other resource types...
    
    if (!canAccess) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    next();
  };
}

// Usage
app.get('/posts/:id',
  authenticateJWT,
  requireResourceAccess('post'),
  async (req, res) => {
    const post = await db.posts.findById(req.params.id);
    res.json(post);
  }
);
```

---

## Rate Limiting

### Token Bucket Algorithm

```typescript
import Redis from 'ioredis';

export class RateLimiter {
  private redis: Redis;
  
  constructor(redis: Redis) {
    this.redis = redis;
  }
  
  async checkLimit(
    key: string,
    maxRequests: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
    const now = Date.now();
    const windowStart = now - (windowSeconds * 1000);
    
    // Use Redis sorted set to track requests
    const pipeline = this.redis.pipeline();
    
    // Remove old requests outside window
    pipeline.zremrangebyscore(key, 0, windowStart);
    
    // Count requests in window
    pipeline.zcard(key);
    
    // Add current request
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    
    // Set expiry
    pipeline.expire(key, windowSeconds);
    
    const results = await pipeline.exec();
    const count = results![1][1] as number;
    
    const allowed = count < maxRequests;
    const remaining = Math.max(0, maxRequests - count - 1);
    const resetAt = new Date(now + (windowSeconds * 1000));
    
    return { allowed, remaining, resetAt };
  }
}

// Middleware
export function rateLimitMiddleware(
  maxRequests: number = 100,
  windowSeconds: number = 60
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // Use IP address or user ID as key
    const key = req.user?.id
      ? `ratelimit:user:${req.user.id}`
      : `ratelimit:ip:${req.ip}`;
    
    const result = await rateLimiter.checkLimit(
      key,
      maxRequests,
      windowSeconds
    );
    
    // Set rate limit headers
    res.setHeader('X-RateLimit-Limit', maxRequests);
    res.setHeader('X-RateLimit-Remaining', result.remaining);
    res.setHeader('X-RateLimit-Reset', result.resetAt.toISOString());
    
    if (!result.allowed) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: result.resetAt,
      });
    }
    
    next();
  };
}

// Usage
app.use('/api', rateLimitMiddleware(100, 60)); // 100 req/min
app.use('/api/auth/login', rateLimitMiddleware(5, 300)); // 5 req/5min
```

### Distributed Rate Limiting

```typescript
export class DistributedRateLimiter {
  private redis: Redis;
  
  async checkLimit(
    key: string,
    maxRequests: number,
    windowMs: number
  ): Promise<RateLimitResult> {
    const script = `
      local key = KEYS[1]
      local max = tonumber(ARGV[1])
      local window = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      -- Remove old entries
      redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
      
      -- Get current count
      local count = redis.call('ZCARD', key)
      
      if count < max then
        -- Add new entry
        redis.call('ZADD', key, now, now .. '-' .. math.random())
        redis.call('EXPIRE', key, math.ceil(window / 1000))
        return {1, max - count - 1, now + window}
      else
        -- Get oldest entry for reset time
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        local reset = tonumber(oldest[2]) + window
        return {0, 0, reset}
      end
    `;
    
    const result = await this.redis.eval(
      script,
      1,
      key,
      maxRequests,
      windowMs,
      Date.now()
    ) as [number, number, number];
    
    return {
      allowed: result[0] === 1,
      remaining: result[1],
      resetAt: new Date(result[2]),
    };
  }
}
```

---

## Input Validation

### Zod Validation

```typescript
import { z } from 'zod';

// Define schemas
export const UserRegistrationSchema = z.object({
  email: z.string().email('Invalid email format'),
  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[a-z]/, 'Password must contain lowercase letter')
    .regex(/[0-9]/, 'Password must contain number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain special character'),
  name: z.string()
    .min(2, 'Name must be at least 2 characters')
    .max(100, 'Name must be at most 100 characters')
    .regex(/^[a-zA-Z\s-']+$/, 'Name contains invalid characters'),
  age: z.number()
    .int('Age must be an integer')
    .min(13, 'Must be at least 13 years old')
    .max(120, 'Invalid age'),
});

export const PostCreationSchema = z.object({
  title: z.string()
    .min(5, 'Title must be at least 5 characters')
    .max(200, 'Title must be at most 200 characters'),
  content: z.string()
    .min(10, 'Content must be at least 10 characters')
    .max(10000, 'Content must be at most 10000 characters'),
  tags: z.array(z.string()).max(10, 'Maximum 10 tags allowed'),
  visibility: z.enum(['public', 'private', 'unlisted']),
});

// Validation middleware
export function validateRequest<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        });
      }
      next(error);
    }
  };
}

// Usage
app.post('/auth/register',
  validateRequest(UserRegistrationSchema),
  async (req, res) => {
    const user = await authService.register(
      req.body.email,
      req.body.password
    );
    res.json(user);
  }
);
```

### SQL Injection Prevention

```typescript
// NEVER DO THIS
const email = req.body.email;
const query = `SELECT * FROM users WHERE email = '${email}'`; // VULNERABLE!

// ALWAYS use parameterized queries
const email = req.body.email;
const result = await pool.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

### XSS Prevention

```typescript
import DOMPurify from 'isomorphic-dompurify';

export function sanitizeHTML(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: [],
  });
}

// Use in endpoints
app.post('/posts',
  authenticateJWT,
  validateRequest(PostCreationSchema),
  async (req, res) => {
    const post = await db.posts.create({
      title: req.body.title,
      content: sanitizeHTML(req.body.content), // Sanitize user input
      authorId: req.user.id,
    });
    res.json(post);
  }
);
```

---

## Secrets Management

### Environment Variables

```typescript
import { z } from 'zod';

// Validate all required secrets on startup
const SecretsSchema = z.object({
  JWT_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  ENCRYPTION_KEY: z.string().length(32),
  AWS_ACCESS_KEY_ID: z.string().optional(),
  AWS_SECRET_ACCESS_KEY: z.string().optional(),
});

export function validateSecrets(): void {
  try {
    SecretsSchema.parse(process.env);
    logger.info('All required secrets validated');
  } catch (error) {
    logger.error({ error }, 'Secret validation failed');
    process.exit(1);
  }
}

// Call on startup
validateSecrets();
```

### AWS Secrets Manager (Production)

```typescript
import {
  SecretsManagerClient,
  GetSecretValueCommand,
} from '@aws-sdk/client-secrets-manager';

export class SecretsManager {
  private client: SecretsManagerClient;
  private cache: Map<string, { value: string; expiresAt: number }>;
  
  constructor() {
    this.client = new SecretsManagerClient({ region: 'us-east-1' });
    this.cache = new Map();
  }
  
  async getSecret(secretName: string): Promise<string> {
    // Check cache
    const cached = this.cache.get(secretName);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }
    
    // Fetch from AWS
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await this.client.send(command);
    
    if (!response.SecretString) {
      throw new Error(`Secret ${secretName} not found`);
    }
    
    // Cache for 5 minutes
    this.cache.set(secretName, {
      value: response.SecretString,
      expiresAt: Date.now() + 5 * 60 * 1000,
    });
    
    return response.SecretString;
  }
  
  async getJSONSecret<T>(secretName: string): Promise<T> {
    const secret = await this.getSecret(secretName);
    return JSON.parse(secret);
  }
}
```

---

## HTTPS & TLS

### TLS Configuration

```typescript
import https from 'https';
import fs from 'fs';

// Production: Use Let's Encrypt or AWS Certificate Manager
const options = {
  key: fs.readFileSync('/etc/letsencrypt/live/example.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/example.com/fullchain.pem'),
  
  // TLS configuration
  minVersion: 'TLSv1.2' as const,
  ciphers: [
    'ECDHE-ECDSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES256-GCM-SHA384',
  ].join(':'),
  honorCipherOrder: true,
};

const server = https.createServer(options, app);
server.listen(443);

// Redirect HTTP to HTTPS
http.createServer((req, res) => {
  res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
  res.end();
}).listen(80);
```

---

## Security Headers

```typescript
import helmet from 'helmet';

app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  
  // HSTS: Force HTTPS
  hsts: {
    maxAge: 31536000, // 1 year
    includeSubDomains: true,
    preload: true,
  },
  
  // Prevent clickjacking
  frameguard: {
    action: 'deny',
  },
  
  // Prevent MIME sniffing
  noSniff: true,
  
  // XSS protection
  xssFilter: true,
  
  // Hide X-Powered-By
  hidePoweredBy: true,
}));

// Additional security headers
app.use((req, res, next) => {
  // Prevent caching of sensitive data
  res.setHeader('Cache-Control', 'no-store');
  res.setHeader('Pragma', 'no-cache');
  
  // Referrer policy
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  
  // Permissions policy
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  
  next();
});
```

---

## CORS Configuration

```typescript
import cors from 'cors';

const corsOptions: cors.CorsOptions = {
  // Allowed origins
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://example.com',
      'https://app.example.com',
    ];
    
    if (process.env.NODE_ENV === 'development') {
      allowedOrigins.push('http://localhost:3000');
    }
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  
  // Allowed methods
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  
  // Allowed headers
  allowedHeaders: ['Content-Type', 'Authorization', 'X-API-Key'],
  
  // Expose headers to client
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
  
  // Allow credentials (cookies, authorization headers)
  credentials: true,
  
  // Preflight cache duration
  maxAge: 86400, // 24 hours
};

app.use(cors(corsOptions));
```

---

## Dependency Security

### npm audit

```bash
# Check for vulnerabilities
npm audit

# Fix automatically fixable issues
npm audit fix

# Force fix (may break things)
npm audit fix --force

# View detailed report
npm audit --json
```

### Automated Scanning

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *' # Daily at midnight

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

---

## Security Monitoring

### Metrics

```typescript
import { Counter, Histogram } from 'prom-client';

export const securityMetrics = {
  authAttempts: new Counter({
    name: 'auth_attempts_total',
    help: 'Total authentication attempts',
    labelNames: ['success', 'method'],
  }),
  
  authFailures: new Counter({
    name: 'auth_failures_total',
    help: 'Failed authentication attempts',
    labelNames: ['reason', 'method'],
  }),
  
  rateLimitHits: new Counter({
    name: 'rate_limit_hits_total',
    help: 'Rate limit violations',
    labelNames: ['endpoint'],
  }),
  
  suspiciousActivity: new Counter({
    name: 'suspicious_activity_total',
    help: 'Suspicious activity detected',
    labelNames: ['type'],
  }),
};

// Usage
authService.on('login', ({ success, method }) => {
  securityMetrics.authAttempts.inc({ success, method });
  
  if (!success) {
    securityMetrics.authFailures.inc({ reason: 'invalid_credentials', method });
  }
});
```

### Audit Logging

```typescript
export async function auditLog(event: AuditEvent): Promise<void> {
  await db.auditLogs.create({
    userId: event.userId,
    action: event.action,
    resource: event.resource,
    resourceId: event.resourceId,
    ipAddress: event.ipAddress,
    userAgent: event.userAgent,
    timestamp: new Date(),
    details: event.details,
  });
  
  logger.info({
    type: 'audit',
    ...event,
  }, 'Audit log');
}

// Usage
await auditLog({
  userId: req.user.id,
  action: 'DELETE',
  resource: 'post',
  resourceId: postId,
  ipAddress: req.ip,
  userAgent: req.get('user-agent'),
  details: { reason: 'spam' },
});
```

---

## Incident Response

### Security Incident Playbook

1. **Detect**: Monitor for anomalies
2. **Contain**: Disable compromised accounts/keys
3. **Investigate**: Audit logs, check for data breach
4. **Remediate**: Patch vulnerabilities, rotate secrets
5. **Document**: Write incident report
6. **Learn**: Update security measures

### Emergency Actions

```typescript
export class SecurityIncidentManager {
  // Immediately lock user account
  async lockAccount(userId: string, reason: string): Promise<void> {
    await db.users.update(userId, {
      locked: true,
      lockedReason: reason,
      lockedAt: new Date(),
    });
    
    // Invalidate all sessions
    await db.refreshTokens.deleteAllForUser(userId);
    
    // Audit log
    await auditLog({
      userId,
      action: 'ACCOUNT_LOCKED',
      resource: 'user',
      resourceId: userId,
      details: { reason },
    });
    
    logger.warn({ userId, reason }, 'Account locked');
  }
  
  // Revoke all API keys
  async revokeAllAPIKeys(userId: string): Promise<void> {
    await db.apiKeys.deleteAllForUser(userId);
    
    logger.warn({ userId }, 'All API keys revoked');
  }
  
  // Force password reset
  async forcePasswordReset(userId: string): Promise<void> {
    await db.users.update(userId, {
      requirePasswordReset: true,
    });
    
    // Send email notification
    await emailService.sendPasswordResetRequired(userId);
  }
}
```

---

## Summary

### Security Checklist

- [ ] JWT authentication implemented
- [ ] Password hashing with bcrypt (12+ rounds)
- [ ] Rate limiting on all endpoints
- [ ] Input validation with Zod
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (sanitize HTML)
- [ ] HTTPS/TLS enabled in production
- [ ] Security headers configured (Helmet)
- [ ] CORS properly configured
- [ ] Secrets stored securely (not in code)
- [ ] npm audit passing
- [ ] Automated security scans
- [ ] Audit logging enabled
- [ ] Incident response playbook ready

---

## Next Steps

1. **Read**: [Intelligent Caching](INTELLIGENT_CACHING.md) for secure session storage
2. **Read**: [Database Layer](DATABASE_LAYER.md) for secure database practices
3. **Implement**: Authentication middleware
4. **Set up**: Rate limiting
5. **Review**: All dependencies for vulnerabilities

---

**Questions?** Check [DevOps Overview](devops.md) or contact #security
