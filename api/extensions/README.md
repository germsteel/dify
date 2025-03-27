# Extensions 扩展系统设计文档

## 整体架构

```mermaid
graph TD
    Extensions[扩展系统] --> Core[核心扩展]
    Extensions --> Storage[存储扩展]
    Extensions --> System[系统扩展]
    Extensions --> Integration[集成扩展]
    
    Core --> C1[数据库]
    Core --> C2[缓存]
    Core --> C3[任务队列]
    
    Storage --> S1[对象存储]
    Storage --> S2[文件系统]
    Storage --> S3[云存储]
    
    System --> Y1[日志]
    System --> Y2[监控]
    System --> Y3[命令]
    
    Integration --> I1[认证]
    Integration --> I2[邮件]
    Integration --> I3[代理]
```

## 扩展模块说明

### 1. 核心扩展

#### ext_database.py
- **功能**: 数据库集成
```python
def init_app(app: Flask):
    """初始化数据库"""
    db.init_app(app)
    
    # 注册数据库命令
    @app.cli.command('init-db')
    def init_db_command():
        """初始化数据库结构"""
        db.create_all()
```

#### ext_redis.py
- **功能**: Redis 缓存集成
```python
def init_app(app: Flask):
    """初始化 Redis"""
    app.redis = Redis(
        host=app.config['REDIS_HOST'],
        port=app.config['REDIS_PORT'],
        db=app.config['REDIS_DB']
    )
```

#### ext_celery.py
- **功能**: Celery 任务队列集成
```python
def init_app(app: Flask):
    """初始化 Celery"""
    celery = Celery(app.name)
    celery.conf.update(app.config['CELERY_CONFIG'])
    
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)
    
    celery.Task = ContextTask
    app.extensions['celery'] = celery
```

### 2. 存储扩展

#### storage/base_storage.py
- **功能**: 基础存储接口
```python
class BaseStorage(ABC):
    """存储基类"""
    
    @abstractmethod
    async def upload_file(self, file_path: str, content: bytes) -> str:
        """上传文件"""
        pass
    
    @abstractmethod
    async def download_file(self, file_path: str) -> bytes:
        """下载文件"""
        pass
```

#### 云存储实现
```python
class S3Storage(BaseStorage):
    """AWS S3 存储实现"""
    
    def __init__(self, config: dict):
        self.client = boto3.client(
            's3',
            aws_access_key_id=config['access_key'],
            aws_secret_access_key=config['secret_key']
        )
        self.bucket = config['bucket']
    
    async def upload_file(self, file_path: str, content: bytes) -> str:
        """上传到 S3"""
        await self.client.put_object(
            Bucket=self.bucket,
            Key=file_path,
            Body=content
        )
        return f"s3://{self.bucket}/{file_path}"
```

### 3. 系统扩展

#### ext_logging.py
- **功能**: 日志系统配置
```python
def init_app(app: Flask):
    """初始化日志"""
    handler = logging.StreamHandler()
    handler.setFormatter(logging.Formatter(
        '[%(asctime)s] %(levelname)s in %(module)s: %(message)s'
    ))
    app.logger.addHandler(handler)
```

#### ext_sentry.py
- **功能**: Sentry 错误追踪
```python
def init_app(app: Flask):
    """初始化 Sentry"""
    if app.config['SENTRY_DSN']:
        sentry_sdk.init(
            dsn=app.config['SENTRY_DSN'],
            integrations=[FlaskIntegration()]
        )
```

### 4. 集成扩展

#### ext_login.py
- **功能**: 用户认证
```python
def init_app(app: Flask):
    """初始化登录管理"""
    login_manager = LoginManager()
    login_manager.init_app(app)
    
    @login_manager.user_loader
    def load_user(user_id):
        return User.query.get(user_id)
```

#### ext_mail.py
- **功能**: 邮件服务
```python
def init_app(app: Flask):
    """初始化邮件服务"""
    mail = Mail()
    mail.init_app(app)
    app.extensions['mail'] = mail
```

## 扩展加载机制

### 1. 应用初始化
```python
def create_app():
    """创建应用"""
    app = Flask(__name__)
    
    # 加载配置
    app.config.from_object('config.Config')
    
    # 初始化扩展
    init_extensions(app)
    
    return app

def init_extensions(app):
    """初始化所有扩展"""
    extensions = [
        ext_database,
        ext_redis,
        ext_celery,
        ext_login,
        ext_mail,
        # ... 其他扩展
    ]
    
    for extension in extensions:
        extension.init_app(app)
```

### 2. 条件加载
```python
def init_optional_extensions(app):
    """初始化可选扩展"""
    if app.config['SENTRY_ENABLED']:
        ext_sentry.init_app(app)
    
    if app.config['METRICS_ENABLED']:
        ext_app_metrics.init_app(app)
```

## 配置管理

### 1. 扩展配置
```python
class Config:
    """应用配置"""
    
    # 数据库配置
    SQLALCHEMY_DATABASE_URI = 'postgresql://user:pass@localhost/db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    
    # Redis 配置
    REDIS_HOST = 'localhost'
    REDIS_PORT = 6379
    REDIS_DB = 0
    
    # 存储配置
    STORAGE_TYPE = 's3'
    STORAGE_CONFIG = {
        's3': {
            'access_key': 'key',
            'secret_key': 'secret',
            'bucket': 'bucket'
        }
    }
```

### 2. 环境变量
```python
def load_env_config(app):
    """从环境变量加载配置"""
    app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
    app.config['DATABASE_URL'] = os.getenv('DATABASE_URL')
    app.config['REDIS_URL'] = os.getenv('REDIS_URL')
```

## 最佳实践

### 1. 扩展开发
- 遵循单一职责
- 提供清晰接口
- 实现优雅降级
- 完善错误处理

### 2. 性能优化
- 懒加载机制
- 资源池化
- 缓存策略
- 异步支持

### 3. 可维护性
- 模块化设计
- 配置外部化
- 完善文档
- 单元测试

### 4. 安全考虑
- 配置验证
- 资源隔离
- 权限控制
- 安全审计

## 开发指南

### 1. 添加新扩展
1. 创建扩展模块
2. 实现初始化方法
3. 添加配置项
4. 编写文档和测试

### 2. 扩展集成
1. 注册扩展
2. 配置参数
3. 初始化顺序
4. 测试验证

### 3. 故障排除
1. 日志记录
2. 状态检查
3. 异常处理
4. 诊断工具
