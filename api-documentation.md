# API Documentation & OpenAPI/Swagger Guide

---
**Status**: 💡 **GUIDANCE** - Best practices for API documentation implementation  
**Priority**: 🟡 **P1** - Essential for team productivity and external integrations  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team  

---

## Overview

API documentation is critical for frontend developers, third-party integrators, and future maintainers. This guide covers OpenAPI/Swagger specification, auto-generation from code, keeping docs in sync, and best practices for maintainable API documentation.

---

## The Problem: Documentation Drift

### Why Manual Documentation Fails

```typescript
// Code reality (current)
app.post('/api/users', async (req, res) => {
  const { email, name, age, phone } = req.body;  // Added phone field
  // ...
});

// Documentation (outdated - missing phone field)
/**
 * POST /api/users
 * Body: { email, name, age }
 */
```

**Result**: Frontend breaks, integrations fail, developers waste time

### The Solution: Single Source of Truth

**Principle**: Documentation should be generated FROM code, not written separately.

---

## OpenAPI/Swagger Specification

### What is OpenAPI?

OpenAPI (formerly Swagger) is a **machine-readable** API specification format that:
- Documents endpoints, parameters, responses
- Enables auto-generated API clients
- Provides interactive API testing (Swagger UI)
- Validates requests/responses
- Generates code in multiple languages

### OpenAPI 3.0 Structure

```yaml
openapi: 3.0.0
info:
  title: My API
  version: 1.2.3
  description: API for my application
  contact:
    name: API Support
    email: api@myapp.com

servers:
  - url: https://api.myapp.com/v1
    description: Production
  - url: https://staging-api.myapp.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: List users
      tags: [Users]
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  users:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  total:
                    type: integer
                  page:
                    type: integer

    post:
      summary: Create user
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    User:
      type: object
      required:
        - id
        - email
        - name
      properties:
        id:
          type: integer
          example: 123
        email:
          type: string
          format: email
          example: user@example.com
        name:
          type: string
          example: John Doe
        age:
          type: integer
          minimum: 0
          maximum: 150
          example: 30
        createdAt:
          type: string
          format: date-time
          example: '2026-01-27T12:00:00Z'

    CreateUserRequest:
      type: object
      required:
        - email
        - name
      properties:
        email:
          type: string
          format: email
        name:
          type: string
        age:
          type: integer
          minimum: 0

    Error:
      type: object
      properties:
        error:
          type: object
          properties:
            message:
              type: string
            code:
              type: string
            details:
              type: object

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

---

## Auto-Generation Strategies

### Strategy 1: Generate from Code (Recommended)

**Pros**: Always in sync, single source of truth  
**Cons**: Requires annotations in code

### Strategy 2: Write OpenAPI First

**Pros**: Design-first approach, can generate code  
**Cons**: Easy to drift from implementation

### Strategy 3: Hybrid Approach

**Pros**: Best of both worlds  
**Cons**: More complex setup

---

## Auto-Generation from Code

### Using TypeScript + Zod + tsoa

```typescript
// Install dependencies
npm install tsoa @tsoa/runtime swagger-ui-express
npm install --save-dev @types/swagger-ui-express

// src/models/user.model.ts
export interface User {
  /** User ID */
  id: number;
  
  /** Email address */
  email: string;
  
  /** Full name */
  name: string;
  
  /** Age in years */
  age?: number;
  
  /** Account creation timestamp */
  createdAt: Date;
}

export interface CreateUserRequest {
  /** Email address (required) */
  email: string;
  
  /** Full name (required) */
  name: string;
  
  /** Age in years (optional) */
  age?: number;
}

export interface UserListResponse {
  users: User[];
  total: number;
  page: number;
}

// src/controllers/user.controller.ts
import { Controller, Get, Post, Body, Query, Route, Tags, Response, Security } from 'tsoa';
import { User, CreateUserRequest, UserListResponse } from '../models/user.model';

@Route('users')
@Tags('Users')
export class UserController extends Controller {
  /**
   * List all users with pagination
   * @param page Page number (default: 1)
   * @param limit Items per page (default: 20, max: 100)
   */
  @Get()
  @Security('jwt')
  @Response<{ error: { message: string } }>(401, 'Unauthorized')
  public async listUsers(
    @Query() page: number = 1,
    @Query() limit: number = 20
  ): Promise<UserListResponse> {
    const users = await userService.listUsers(page, limit);
    return users;
  }

  /**
   * Create a new user
   * @param requestBody User data
   */
  @Post()
  @Security('jwt')
  @Response<User>(201, 'Created')
  @Response<{ error: { message: string; code: string } }>(400, 'Validation error')
  @Response<{ error: { message: string } }>(401, 'Unauthorized')
  public async createUser(
    @Body() requestBody: CreateUserRequest
  ): Promise<User> {
    this.setStatus(201);
    const user = await userService.createUser(requestBody);
    return user;
  }

  /**
   * Get user by ID
   * @param userId User ID
   */
  @Get('{userId}')
  @Security('jwt')
  @Response<User>(200, 'Success')
  @Response<{ error: { message: string } }>(404, 'User not found')
  public async getUser(
    userId: number
  ): Promise<User> {
    const user = await userService.getUser(userId);
    if (!user) {
      throw new Error('User not found');
    }
    return user;
  }
}

// tsoa.json - Configuration
{
  "entryFile": "src/server.ts",
  "noImplicitAdditionalProperties": "throw-on-extras",
  "controllerPathGlobs": ["src/controllers/**/*.controller.ts"],
  "spec": {
    "outputDirectory": "docs",
    "specVersion": 3,
    "name": "My API",
    "version": "1.2.3",
    "description": "API Documentation",
    "contact": {
      "name": "API Support",
      "email": "api@myapp.com"
    },
    "securityDefinitions": {
      "jwt": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    }
  },
  "routes": {
    "routesDir": "src/routes"
  }
}

// package.json scripts
{
  "scripts": {
    "generate:docs": "tsoa spec-and-routes",
    "dev": "npm run generate:docs && tsx watch src/server.ts",
    "build": "npm run generate:docs && tsc"
  }
}
```

### Using Zod + OpenAPI Generator

```typescript
// Install dependencies
npm install zod zod-to-openapi express-zod-api

// src/schemas/user.schema.ts
import { z } from 'zod';
import { extendZodWithOpenApi } from 'zod-to-openapi';

extendZodWithOpenApi(z);

export const UserSchema = z.object({
  id: z.number().int().positive().openapi({ example: 123 }),
  email: z.string().email().openapi({ example: 'user@example.com' }),
  name: z.string().min(1).max(100).openapi({ example: 'John Doe' }),
  age: z.number().int().min(0).max(150).optional().openapi({ example: 30 }),
  createdAt: z.string().datetime().openapi({ example: '2026-01-27T12:00:00Z' })
}).openapi('User');

export const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150).optional()
}).openapi('CreateUserRequest');

export const UserListResponseSchema = z.object({
  users: z.array(UserSchema),
  total: z.number().int(),
  page: z.number().int()
}).openapi('UserListResponse');

// src/routes/users.ts
import { createRoute, OpenAPIRegistry } from '@asteasolutions/zod-to-openapi';
import { UserSchema, CreateUserSchema, UserListResponseSchema } from '../schemas/user.schema';

const registry = new OpenAPIRegistry();

// List users route
registry.registerPath({
  method: 'get',
  path: '/users',
  tags: ['Users'],
  summary: 'List users',
  security: [{ BearerAuth: [] }],
  request: {
    query: z.object({
      page: z.number().int().min(1).default(1),
      limit: z.number().int().min(1).max(100).default(20)
    })
  },
  responses: {
    200: {
      description: 'Success',
      content: {
        'application/json': {
          schema: UserListResponseSchema
        }
      }
    },
    401: {
      description: 'Unauthorized'
    }
  }
});

// Create user route
registry.registerPath({
  method: 'post',
  path: '/users',
  tags: ['Users'],
  summary: 'Create user',
  security: [{ BearerAuth: [] }],
  request: {
    body: {
      content: {
        'application/json': {
          schema: CreateUserSchema
        }
      }
    }
  },
  responses: {
    201: {
      description: 'Created',
      content: {
        'application/json': {
          schema: UserSchema
        }
      }
    },
    400: {
      description: 'Validation error'
    }
  }
});

// Generate OpenAPI spec
import { OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi';

const generator = new OpenApiGeneratorV3(registry.definitions);

const openApiDoc = generator.generateDocument({
  openapi: '3.0.0',
  info: {
    title: 'My API',
    version: '1.2.3',
    description: 'API Documentation'
  },
  servers: [
    {
      url: 'https://api.myapp.com/v1',
      description: 'Production'
    }
  ]
});

// Export to file
import fs from 'fs';
fs.writeFileSync('docs/openapi.json', JSON.stringify(openApiDoc, null, 2));
```

### Using JSDoc Comments (Simple Approach)

```typescript
// Install swagger-jsdoc
npm install swagger-jsdoc swagger-ui-express

// src/routes/users.ts
/**
 * @openapi
 * /users:
 *   get:
 *     tags:
 *       - Users
 *     summary: List users
 *     parameters:
 *       - name: page
 *         in: query
 *         schema:
 *           type: integer
 *           default: 1
 *       - name: limit
 *         in: query
 *         schema:
 *           type: integer
 *           default: 20
 *     responses:
 *       200:
 *         description: Success
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 users:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 total:
 *                   type: integer
 */
app.get('/users', async (req, res) => {
  // Implementation
});

/**
 * @openapi
 * /users:
 *   post:
 *     tags:
 *       - Users
 *     summary: Create user
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreateUserRequest'
 *     responses:
 *       201:
 *         description: Created
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/User'
 */
app.post('/users', async (req, res) => {
  // Implementation
});

// src/swagger.ts
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'My API',
      version: '1.2.3',
      description: 'API Documentation'
    },
    servers: [
      {
        url: 'https://api.myapp.com/v1',
        description: 'Production'
      }
    ],
    components: {
      schemas: {
        User: {
          type: 'object',
          properties: {
            id: { type: 'integer' },
            email: { type: 'string', format: 'email' },
            name: { type: 'string' },
            age: { type: 'integer' }
          }
        },
        CreateUserRequest: {
          type: 'object',
          required: ['email', 'name'],
          properties: {
            email: { type: 'string', format: 'email' },
            name: { type: 'string' },
            age: { type: 'integer' }
          }
        }
      },
      securitySchemes: {
        BearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        }
      }
    },
    security: [{ BearerAuth: [] }]
  },
  apis: ['./src/routes/**/*.ts'] // Path to route files
};

export const swaggerSpec = swaggerJsdoc(options);

// src/server.ts
import { swaggerSpec } from './swagger';

// Serve swagger.json
app.get('/api-docs.json', (req, res) => {
  res.json(swaggerSpec);
});

// Serve Swagger UI
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
```

---

## Keeping Documentation in Sync

### 1. Auto-Generate on Build (Recommended)

```json
// package.json
{
  "scripts": {
    "prebuild": "npm run generate:docs",
    "generate:docs": "tsoa spec-and-routes",
    "build": "tsc",
    "predev": "npm run generate:docs",
    "dev": "tsx watch src/server.ts"
  }
}
```

### 2. Git Pre-Commit Hook

```bash
#!/bin/bash
# .husky/pre-commit

# Regenerate API docs
npm run generate:docs

# Add generated files to commit
git add docs/openapi.json
git add docs/swagger.json

echo "✅ API documentation regenerated"
```

### 3. CI/CD Check

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate API docs
        run: npm run generate:docs
      
      - name: Check for uncommitted changes
        run: |
          if [[ -n $(git status --porcelain docs/) ]]; then
            echo "❌ API documentation is out of sync!"
            echo "Run 'npm run generate:docs' and commit the changes"
            git diff docs/
            exit 1
          fi
          echo "✅ API documentation is in sync"
      
      - name: Validate OpenAPI spec
        run: npx @redocly/cli lint docs/openapi.json
```

### 4. Automated PR Comments

```yaml
# .github/workflows/api-docs-diff.yml
name: API Docs Diff

on: pull_request

jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Check API changes
        id: diff
        run: |
          git diff origin/main docs/openapi.json > api-diff.txt
          if [[ -s api-diff.txt ]]; then
            echo "has_changes=true" >> $GITHUB_OUTPUT
          fi
      
      - name: Comment on PR
        if: steps.diff.outputs.has_changes == 'true'
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '⚠️ This PR contains API changes. Please review the API documentation updates.'
            })
```

---

## Validation & Testing

### Validate OpenAPI Spec

```bash
# Install validator
npm install --save-dev @redocly/cli

# Validate spec
npx @redocly/cli lint docs/openapi.json

# Or add to package.json
{
  "scripts": {
    "validate:api": "redocly lint docs/openapi.json"
  }
}
```

### Test API Against Spec

```typescript
// tests/api-spec.test.ts
import { describe, it, expect } from '@jest/globals';
import request from 'supertest';
import { app } from '../src/server';
import OpenAPIValidator from 'express-openapi-validator';

describe('API Spec Validation', () => {
  it('should validate responses against OpenAPI spec', async () => {
    // Add validator middleware
    app.use(
      OpenAPIValidator.middleware({
        apiSpec: './docs/openapi.json',
        validateRequests: true,
        validateResponses: true
      })
    );
    
    // Test endpoint
    const response = await request(app)
      .get('/users')
      .expect(200);
    
    // If this passes, response matches spec
    expect(response.body).toHaveProperty('users');
    expect(response.body).toHaveProperty('total');
  });
});
```

### Runtime Request/Response Validation

```typescript
// src/middleware/validation.ts
import OpenAPIValidator from 'express-openapi-validator';

export const apiValidator = OpenAPIValidator.middleware({
  apiSpec: './docs/openapi.json',
  validateRequests: true,    // Validate incoming requests
  validateResponses: true,   // Validate outgoing responses
  validateSecurity: true,    // Validate security requirements
  $refParser: {
    mode: 'dereference'
  }
});

// src/server.ts
import { apiValidator } from './middleware/validation';

// Apply validator after routes are defined
app.use(apiValidator);

// Error handler for validation errors
app.use((err, req, res, next) => {
  if (err.status === 400 && err.errors) {
    // Validation error
    logger.warn('API validation error', {
      path: req.path,
      errors: err.errors
    });
    
    return res.status(400).json({
      error: {
        message: 'Validation failed',
        details: err.errors
      }
    });
  }
  
  next(err);
});
```

---

## Documentation UI

### Swagger UI (Interactive)

```typescript
// src/server.ts
import swaggerUi from 'swagger-ui-express';
import openApiDoc from '../docs/openapi.json';

// Serve Swagger UI
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(openApiDoc, {
  customCss: '.swagger-ui .topbar { display: none }',
  customSiteTitle: 'My API Documentation'
}));

// Visit: http://localhost:3000/api-docs
```

### ReDoc (Clean, Modern)

```typescript
// Install redoc
npm install redoc-express

// src/server.ts
import { serve as redocServe, setup as redocSetup } from 'redoc-express';

// Serve ReDoc
app.use('/api-docs/redoc', 
  redocServe({ 
    specUrl: '/api-docs.json',
    title: 'My API Documentation'
  })
);

// Visit: http://localhost:3000/api-docs/redoc
```

### Both Swagger UI + ReDoc

```typescript
// src/server.ts
import swaggerUi from 'swagger-ui-express';
import { serve as redocServe } from 'redoc-express';
import openApiDoc from '../docs/openapi.json';

// Serve raw JSON
app.get('/api-docs.json', (req, res) => {
  res.json(openApiDoc);
});

// Serve Swagger UI (interactive testing)
app.use('/api-docs/swagger', 
  swaggerUi.serve, 
  swaggerUi.setup(openApiDoc)
);

// Serve ReDoc (clean reading)
app.use('/api-docs/redoc', 
  redocServe({ specUrl: '/api-docs.json' })
);

// Landing page
app.get('/api-docs', (req, res) => {
  res.send(`
    <html>
      <head><title>API Documentation</title></head>
      <body>
        <h1>API Documentation</h1>
        <ul>
          <li><a href="/api-docs/swagger">Swagger UI (Interactive)</a></li>
          <li><a href="/api-docs/redoc">ReDoc (Clean Reading)</a></li>
          <li><a href="/api-docs.json">OpenAPI JSON</a></li>
        </ul>
      </body>
    </html>
  `);
});
```

---

## Documentation Best Practices

### 1. Comprehensive Examples

```yaml
# Good: Includes examples
User:
  type: object
  properties:
    email:
      type: string
      format: email
      example: 'user@example.com'
    age:
      type: integer
      minimum: 0
      maximum: 150
      example: 30

# Bad: No examples
User:
  type: object
  properties:
    email:
      type: string
    age:
      type: integer
```

### 2. Descriptive Error Responses

```yaml
responses:
  '400':
    description: Validation error
    content:
      application/json:
        schema:
          type: object
          properties:
            error:
              type: object
              properties:
                message:
                  type: string
                  example: 'Validation failed'
                code:
                  type: string
                  example: 'VALIDATION_ERROR'
                details:
                  type: array
                  items:
                    type: object
                    properties:
                      field:
                        type: string
                        example: 'email'
                      message:
                        type: string
                        example: 'Invalid email format'
```

### 3. Document Authentication

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT token obtained from /auth/login endpoint.
        
        Example:
        ```
        Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
        ```

security:
  - BearerAuth: []
```

### 4. Version Documentation

```yaml
info:
  title: My API
  version: 1.2.3
  description: |
    # My API Documentation
    
    ## Versioning
    
    This API uses semantic versioning (MAJOR.MINOR.PATCH).
    
    - **MAJOR**: Breaking changes
    - **MINOR**: New features (backward compatible)
    - **PATCH**: Bug fixes
    
    ## Changelog
    
    ### v1.2.3 (2026-01-27)
    - Added user export endpoint
    - Fixed pagination bug
    
    ### v1.2.0 (2026-01-20)
    - Added authentication
    - Added user management
```

### 5. Document Rate Limits

```yaml
paths:
  /users:
    get:
      summary: List users
      description: |
        Lists all users with pagination.
        
        **Rate Limit**: 100 requests per 15 minutes per IP address
        
        Rate limit headers:
        - `X-RateLimit-Limit`: Maximum requests allowed
        - `X-RateLimit-Remaining`: Requests remaining
        - `X-RateLimit-Reset`: Unix timestamp when limit resets
```

---

## Documentation Checklist

### Pre-Release Documentation Checklist

- [ ] All endpoints documented
- [ ] All request parameters documented with types
- [ ] All response schemas documented
- [ ] All error responses documented
- [ ] Examples provided for all schemas
- [ ] Authentication/authorization documented
- [ ] Rate limits documented
- [ ] Versioning strategy documented
- [ ] Changelog up to date
- [ ] OpenAPI spec validates without errors
- [ ] Swagger UI accessible
- [ ] Documentation automatically regenerated on build
- [ ] CI/CD checks documentation is in sync

### Maintenance Checklist

- [ ] Update docs when adding endpoints
- [ ] Update docs when changing request/response format
- [ ] Update docs when changing authentication
- [ ] Update examples when schema changes
- [ ] Deprecate old endpoints properly
- [ ] Document breaking changes in changelog
- [ ] Version API appropriately

---

## Common Patterns

### Pagination

```yaml
components:
  parameters:
    PageParam:
      name: page
      in: query
      schema:
        type: integer
        minimum: 1
        default: 1
      description: Page number
    
    LimitParam:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
      description: Items per page

  schemas:
    PaginatedResponse:
      type: object
      properties:
        data:
          type: array
          items: {}
        pagination:
          type: object
          properties:
            total:
              type: integer
              description: Total number of items
            page:
              type: integer
              description: Current page
            limit:
              type: integer
              description: Items per page
            totalPages:
              type: integer
              description: Total number of pages

paths:
  /users:
    get:
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
      responses:
        '200':
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/PaginatedResponse'
                  - type: object
                    properties:
                      data:
                        type: array
                        items:
                          $ref: '#/components/schemas/User'
```

### Filtering & Sorting

```yaml
components:
  parameters:
    SortParam:
      name: sort
      in: query
      schema:
        type: string
        enum: [name, email, createdAt, -name, -email, -createdAt]
      description: |
        Sort field and direction.
        Prefix with '-' for descending order.
        Examples: 'name', '-createdAt'
    
    FilterParam:
      name: filter
      in: query
      style: deepObject
      explode: true
      schema:
        type: object
        properties:
          status:
            type: string
            enum: [active, inactive, pending]
          role:
            type: string
            enum: [admin, user, guest]
          createdAfter:
            type: string
            format: date-time
```

### Bulk Operations

```yaml
paths:
  /users/bulk:
    post:
      summary: Create multiple users
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                users:
                  type: array
                  items:
                    $ref: '#/components/schemas/CreateUserRequest'
                  minItems: 1
                  maxItems: 100
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                type: object
                properties:
                  created:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  failed:
                    type: array
                    items:
                      type: object
                      properties:
                        index:
                          type: integer
                        error:
                          type: string
```

---

## Deployment & Access Control

### Production Documentation Access

```typescript
// src/server.ts
import { isProduction } from './config/environment';

if (!isProduction) {
  // Only serve docs in non-production
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(openApiDoc));
} else {
  // In production, require authentication
  app.use('/api-docs', requireAuth, requireRole('admin'), 
    swaggerUi.serve, swaggerUi.setup(openApiDoc)
  );
}

// Or host on separate domain
// docs.myapp.com (public)
// api.myapp.com (no docs endpoint)
```

### Static Documentation Site

```bash
# Generate static HTML from OpenAPI
npx redoc-cli bundle docs/openapi.json -o docs/index.html

# Deploy to S3/Netlify/Vercel
aws s3 sync docs/ s3://api-docs.myapp.com/

# Or use GitHub Pages
# Commit docs/index.html to gh-pages branch
```

---

## Tools & Resources

### Documentation Tools

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| **Swagger UI** | Interactive testing | Try endpoints in browser | Cluttered for large APIs |
| **ReDoc** | Clean reading | Beautiful, responsive | No testing capability |
| **Postman** | API testing | Collection sharing | Not from code |
| **Stoplight** | Design-first | Full featured | Paid for teams |
| **ReadMe** | Hosted docs | Professional, analytics | Paid service |

### Validation Tools

```bash
# OpenAPI spec validators
npm install --save-dev @redocly/cli
npx redocly lint openapi.json

# Runtime validation
npm install express-openapi-validator

# Testing
npm install openapi-backend jest-openapi
```

---

## Quick Start Guide

### Minimum Viable Documentation

```typescript
// 1. Install dependencies
npm install swagger-jsdoc swagger-ui-express

// 2. Create swagger.ts
import swaggerJsdoc from 'swagger-jsdoc';

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'My API',
      version: '1.0.0'
    }
  },
  apis: ['./src/routes/**/*.ts']
};

export const swaggerSpec = swaggerJsdoc(options);

// 3. Add JSDoc comments to routes
/**
 * @openapi
 * /users:
 *   get:
 *     summary: List users
 *     responses:
 *       200:
 *         description: Success
 */
app.get('/users', async (req, res) => { /* ... */ });

// 4. Serve documentation
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));

// 5. Visit http://localhost:3000/api-docs
```

---

## Summary

### The Golden Rule

**Code is the source of truth. Documentation is generated FROM code.**

### Key Principles

1. **Auto-generate** documentation from code
2. **Validate** docs in CI/CD pipeline
3. **Regenerate** on every build
4. **Test** API responses against spec
5. **Version** API properly
6. **Provide examples** for everything
7. **Keep it simple** - start minimal, expand as needed

### Recommended Stack

- **TypeScript + Zod** for type-safe schemas
- **tsoa** or **zod-to-openapi** for auto-generation
- **Swagger UI** for interactive testing
- **ReDoc** for clean reading
- **OpenAPI Validator** for runtime validation
- **Redocly CLI** for spec validation

### Time Investment

- **Initial setup**: 2-4 hours
- **Per endpoint**: 5-10 minutes (with auto-generation)
- **Maintenance**: Automatic (regenerates on build)

### Related Documentation
- [API Versioning](./API_VERSIONING.md)
- [Authentication](./AUTHENTICATION.md)
- [Error Handling](./ERROR_HANDLING.md)

---

**Last Updated**: 2026-01-27  
**Owner**: API Team  
**Review**: On major API changes