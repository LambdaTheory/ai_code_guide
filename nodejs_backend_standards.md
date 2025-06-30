# Node.js 后端服务开发规范

## 1. 项目结构规范

```
project-root/
├── src/
│   ├── controllers/          # 控制器层
│   ├── services/            # 业务逻辑层
│   ├── models/              # 数据模型
│   ├── middleware/          # 中间件
│   ├── routes/              # 路由定义
│   ├── utils/               # 工具函数
│   ├── config/              # 配置文件
│   ├── validators/          # 数据验证
│   └── types/               # TypeScript类型定义
├── tests/                   # 测试文件
├── docs/                    # 文档
├── scripts/                 # 脚本文件
├── .env.example            # 环境变量示例
├── .gitignore
├── package.json
├── tsconfig.json           # TypeScript配置
└── README.md
```

## 2. 代码规范

### 2.1 命名规范
- **文件名**: 使用kebab-case (user-service.ts)
- **变量/函数**: 使用camelCase (getUserById)
- **常量**: 使用UPPER_SNAKE_CASE (MAX_RETRY_COUNT)
- **类名**: 使用PascalCase (UserService)
- **接口**: 使用PascalCase，以I开头 (IUserRepository)

### 2.2 TypeScript使用
```typescript
// 严格类型定义
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
}

// 使用泛型
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
}
```

## 3. 架构设计规范

### 3.1 分层架构
```typescript
// Controller层 - 处理HTTP请求
export class UserController {
  constructor(private userService: UserService) {}
  
  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.params.id;
      const user = await this.userService.getUserById(userId);
      res.json({ success: true, data: user });
    } catch (error) {
      res.status(500).json({ success: false, error: error.message });
    }
  }
}

// Service层 - 业务逻辑
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async getUserById(id: string): Promise<User> {
    if (!id) {
      throw new Error('User ID is required');
    }
    return await this.userRepository.findById(id);
  }
}

// Repository层 - 数据访问
export class UserRepository {
  constructor(private supabase: SupabaseClient) {}
  
  async findById(id: string): Promise<User | null> {
    const { data, error } = await this.supabase
      .from('users')
      .select('*')
      .eq('id', id)
      .single();
    
    if (error) throw error;
    return data;
  }
}
```

## 4. Supabase集成规范

### 4.1 连接配置
```typescript
// config/supabase.ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.SUPABASE_URL!;
const supabaseKey = process.env.SUPABASE_ANON_KEY!;

export const supabase = createClient(supabaseUrl, supabaseKey, {
  auth: {
    autoRefreshToken: true,
    persistSession: false
  }
});
```

### 4.2 数据库操作规范
```typescript
// 使用事务
async createUserWithProfile(userData: CreateUserData): Promise<User> {
  const { data, error } = await this.supabase.rpc('create_user_with_profile', {
    user_data: userData
  });
  
  if (error) {
    throw new DatabaseError(`Failed to create user: ${error.message}`);
  }
  
  return data;
}

// 错误处理
async getUserById(id: string): Promise<User> {
  try {
    const { data, error } = await this.supabase
      .from('users')
      .select('*')
      .eq('id', id)
      .single();
    
    if (error) {
      if (error.code === 'PGRST116') {
        throw new NotFoundError('User not found');
      }
      throw new DatabaseError(error.message);
    }
    
    return data;
  } catch (error) {
    this.logger.error('Database query failed', { id, error });
    throw error;
  }
}
```

## 5. 错误处理规范

### 5.1 自定义错误类
```typescript
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code?: string
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

export class NotFoundError extends AppError {
  constructor(message: string) {
    super(message, 404, 'NOT_FOUND');
  }
}
```

### 5.2 全局错误处理中间件
```typescript
export const errorHandler = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void => {
  if (error instanceof AppError) {
    res.status(error.statusCode).json({
      success: false,
      error: error.message,
      code: error.code
    });
    return;
  }
  
  // 记录未知错误
  logger.error('Unhandled error', { error: error.message, stack: error.stack });
  
  res.status(500).json({
    success: false,
    error: 'Internal server error'
  });
};
```

## 6. 数据验证规范

### 6.1 使用Joi或Zod进行验证
```typescript
import Joi from 'joi';

export const createUserSchema = Joi.object({
  email: Joi.string().email().required(),
  name: Joi.string().min(2).max(50).required(),
  password: Joi.string().min(8).required()
});

// 验证中间件
export const validateRequest = (schema: Joi.ObjectSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error } = schema.validate(req.body);
    if (error) {
      return res.status(400).json({
        success: false,
        error: error.details[0].message
      });
    }
    next();
  };
};
```

## 7. 安全规范

### 7.1 环境变量管理
```typescript
// config/env.ts
import dotenv from 'dotenv';

dotenv.config();

export const config = {
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  supabase: {
    url: process.env.SUPABASE_URL!,
    anonKey: process.env.SUPABASE_ANON_KEY!,
    serviceKey: process.env.SUPABASE_SERVICE_ROLE_KEY!
  },
  jwt: {
    secret: process.env.JWT_SECRET!,
    expiresIn: process.env.JWT_EXPIRES_IN || '24h'
  }
};
```

### 7.2 安全中间件
```typescript
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';

// 安全头部
app.use(helmet());

// CORS配置
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));

// 速率限制
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分钟
  max: 100, // 限制每个IP 100次请求
  message: 'Too many requests from this IP'
});
app.use('/api/', limiter);
```

## 8. 日志规范

### 8.1 结构化日志
```typescript
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// 使用示例
logger.info('User created', { userId: user.id, email: user.email });
logger.error('Database connection failed', { error: error.message });
```

## 9. 测试规范

### 9.1 单元测试
```typescript
// tests/services/user.service.test.ts
import { UserService } from '../../src/services/user.service';
import { UserRepository } from '../../src/repositories/user.repository';

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepository: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockUserRepository = {
      findById: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn()
    } as any;
    
    userService = new UserService(mockUserRepository);
  });

  describe('getUserById', () => {
    it('should return user when found', async () => {
      const mockUser = { id: '1', email: 'test@example.com', name: 'Test User' };
      mockUserRepository.findById.mockResolvedValue(mockUser);

      const result = await userService.getUserById('1');

      expect(result).toEqual(mockUser);
      expect(mockUserRepository.findById).toHaveBeenCalledWith('1');
    });

    it('should throw error when user not found', async () => {
      mockUserRepository.findById.mockResolvedValue(null);

      await expect(userService.getUserById('1')).rejects.toThrow('User not found');
    });
  });
});
```

## 10. API设计规范

### 10.1 RESTful API设计
```typescript
// routes/users.ts
import { Router } from 'express';
import { UserController } from '../controllers/user.controller';
import { validateRequest } from '../middleware/validation';
import { createUserSchema, updateUserSchema } from '../validators/user.validator';

const router = Router();
const userController = new UserController();

// GET /api/users - 获取用户列表
router.get('/', userController.getUsers);

// GET /api/users/:id - 获取单个用户
router.get('/:id', userController.getUserById);

// POST /api/users - 创建用户
router.post('/', validateRequest(createUserSchema), userController.createUser);

// PUT /api/users/:id - 更新用户
router.put('/:id', validateRequest(updateUserSchema), userController.updateUser);

// DELETE /api/users/:id - 删除用户
router.delete('/:id', userController.deleteUser);

export default router;
```

### 10.2 统一响应格式
```typescript
interface ApiResponse<T = any> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
  pagination?: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

// 成功响应
export const successResponse = <T>(data: T, message?: string): ApiResponse<T> => ({
  success: true,
  data,
  message
});

// 错误响应
export const errorResponse = (error: string, message?: string): ApiResponse => ({
  success: false,
  error,
  message
});
```

## 11. 性能优化规范

### 11.1 数据库查询优化
```typescript
// 使用索引
await supabase
  .from('users')
  .select('id, name, email')  // 只选择需要的字段
  .eq('status', 'active')
  .order('created_at', { ascending: false })
  .range(0, 9);  // 分页

// 批量操作
await supabase
  .from('users')
  .upsert(users, { onConflict: 'email' });
```

### 11.2 缓存策略
```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

export class CacheService {
  async get<T>(key: string): Promise<T | null> {
    const cached = await redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }

  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    await redis.setex(key, ttl, JSON.stringify(value));
  }

  async del(key: string): Promise<void> {
    await redis.del(key);
  }
}
```

## 12. 部署和监控规范

### 12.1 健康检查端点
```typescript
// routes/health.ts
router.get('/health', async (req, res) => {
  try {
    // 检查数据库连接
    const { data, error } = await supabase.from('users').select('count').limit(1);
    
    if (error) throw error;

    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        database: 'connected',
        cache: 'connected'
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});
```

### 12.2 环境配置
```typescript
// 不同环境的配置
const configs = {
  development: {
    logLevel: 'debug',
    corsOrigins: ['http://localhost:3000'],
    rateLimit: { windowMs: 15 * 60 * 1000, max: 1000 }
  },
  production: {
    logLevel: 'error',
    corsOrigins: process.env.ALLOWED_ORIGINS?.split(','),
    rateLimit: { windowMs: 15 * 60 * 1000, max: 100 }
  }
};
```

## 13. 代码质量工具配置

### 13.1 ESLint配置
```json
{
  "extends": [
    "@typescript-eslint/recommended",
    "prettier"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "prefer-const": "error",
    "no-var": "error"
  }
}
```

### 13.2 Prettier配置
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
```

## 总结

这套开发规范涵盖了Node.js后端服务开发的各个方面，特别针对Supabase数据库的使用场景。遵循这些规范可以确保代码的可维护性、可扩展性和安全性。记住要根据项目的具体需求适当调整这些规范。
