---
paths:
  - "**/*.js"
  - "**/*.jsx"
  - "**/*.ts"
  - "**/*.tsx"
---

You are Claude, an expert Node.js backend developer with deep knowledge of Express, TypeScript, and database patterns.

## Project Structure
```
src/
├── config/          # Configuration
├── controllers/     # Request handlers
├── middleware/      # Express middleware
├── models/          # Data models
├── routes/          # Route definitions
├── services/        # Business logic
├── types/           # TypeScript types
├── utils/           # Utilities
└── app.ts           # Express app
```

## Express App
```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';

export function createApp() {
  const app = express();
  
  // Security
  app.use(helmet());
  app.use(cors({ origin: process.env.CORS_ORIGIN }));
  
  // Parsing
  app.use(express.json({ limit: '10mb' }));
  
  // Routes
  app.use('/api/users', userRoutes);
  app.use('/api/posts', postRoutes);
  
  // Error handler
  app.use(errorHandler);
  
  return app;
}
```

## Controller
```typescript
export class UserController {
  constructor(private userService: UserService) {}
  
  getUser = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.findById(req.params.id);
      
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      
      res.json({ data: user });
    } catch (error) {
      next(error);
    }
  };
  
  createUser = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.create(req.body);
      res.status(201).json({ data: user });
    } catch (error) {
      next(error);
    }
  };
}
```

## Error Handling
```typescript
export class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
  ) {
    super(message);
  }
}

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message });
  }
  
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
}
```