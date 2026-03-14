---
title: "FastAPIで爆速API開発 — Claude Code×Python実践ガイド"
emoji: "📝"
type: "tech"
topics: ["AI", "プログラミング", "エンジニア"]
published: true
---

「APIを1本作るのに半日かかる」——そんな時代は終わりました。Claude Codeを活用すれば、認証付きのプロダクションレベルAPIを1時間で構築できます。

この記事では、FastAPI × Claude Codeで実際に20,000行規模のAPIサーバーを構築・運用している経験から、爆速で高品質なAPI開発を実現する方法を解説します。

## この記事でわかること

- FastAPI + Claude Codeで開発速度が劇的に上がる理由
- プロダクションレベルのAPI設計パターン
- 認証・テスト・セキュリティの実装テンプレート
- 実際のコードで見る「薄いルーター」パターン

## 目次

1. なぜFastAPI × Claude Codeが最強なのか
2. プロジェクト構成のベストプラクティス
3. 薄いルーターパターン
4. 認証の実装（JWT + リフレッシュトークン）
5. テスト戦略（セキュリティテスト必須）
6. ミドルウェアスタックの設計
7. データベース設計とマイグレーション
8. デプロイメント
9. まとめ

---

## 1. なぜFastAPI × Claude Codeが最強なのか

FastAPIはPydanticによる型安全性、自動ドキュメント生成、async対応が特徴です。これらの特徴がClaude Codeと相性抜群です。

**型情報がコンテキストになる:**

```python
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)
    display_name: str = Field(max_length=100)
```

この型定義があるだけで、Claude Codeはバリデーション、テスト、ドキュメントを正確に生成できます。型がないPythonコードでは、こうはいきません。

**自動ドキュメントがレビュー基盤になる:**

FastAPIは`/docs`（Swagger UI）と`/redoc`を自動生成します。Claude Codeが新しいエンドポイントを追加したら、`/docs`を開いてすぐに動作確認ができます。

---

## 2. プロジェクト構成のベストプラクティス

Claude Codeに最初に教えるべきはプロジェクト構成です。`CLAUDE.md`に以下を書きます：

```
# API Architecture

## Structure
services/api/app/
├── main.py          # FastAPI app + middleware stack
├── core/            # Shared infrastructure
│   ├── config.py    # Pydantic BaseSettings
│   ├── database.py  # Async SQLAlchemy
│   ├── security.py  # JWT utilities
│   └── tenant.py    # Multi-tenant isolation
├── models/          # SQLAlchemy ORM models (<80 lines each)
├── routers/         # Thin routers (<300 lines each)
├── services/        # Business logic (extracted from fat routers)
└── tests/           # pytest, integration-focused
```

この構成をCLAUDE.mdに書いておくと、Claude Codeは新しいエンドポイントを追加する際に自動的にこのパターンに従います。

---

ここから先は有料エリアです。薄いルーターパターンの実装、JWT認証テンプレート、セキュリティテスト戦略など、プロダクションで使える実践的なコードをお伝えします。

---

## 3. 薄いルーターパターン

FastAPIでよくある失敗は「ルーターが肥大化する」ことです。

### ❌ よくある失敗パターン（Fat Router）

```python
@router.post("/workflows")
async def create_workflow(
    data: WorkflowCreate,
    db: AsyncSession = Depends(get_db),
    request: Request = None,
):
    tenant_id = get_required_tenant_id(request)

    # ↓ ビジネスロジックがルーターに直書き（600行以上になりがち）
    existing = await db.execute(
        select(Workflow).where(
            Workflow.tenant_id == tenant_id,
            Workflow.name == data.name
        )
    )
    if existing.scalar_one_or_none():
        raise HTTPException(409, "Workflow already exists")

    workflow = Workflow(
        tenant_id=tenant_id,
        name=data.name,
        # ...30行のマッピングコード
    )
    db.add(workflow)
    await db.commit()
    # ...さらに通知、ログ、後処理...
```

### ✅ 推奨パターン（Thin Router + Service）

```python
# routers/workflows.py（薄い — 入出力の変換のみ）
@router.post("/workflows", response_model=WorkflowResponse)
async def create_workflow(
    data: WorkflowCreate,
    db: AsyncSession = Depends(get_db),
    request: Request = None,
):
    tenant_id = get_required_tenant_id(request)
    workflow = await workflow_service.create(db, tenant_id, data)
    return WorkflowResponse.model_validate(workflow)


# services/workflow_service.py（ビジネスロジックはここ）
async def create(
    db: AsyncSession,
    tenant_id: str,
    data: WorkflowCreate,
) -> Workflow:
    existing = await _find_by_name(db, tenant_id, data.name)
    if existing:
        raise HTTPException(409, "Workflow already exists")

    workflow = Workflow(tenant_id=tenant_id, **data.model_dump())
    db.add(workflow)
    await db.commit()
    await db.refresh(workflow)
    return workflow
```

**ルール:**
- ルーターは300行以下
- サービスは400行以下
- ルーターの仕事: リクエスト解析 → サービス呼び出し → レスポンス返却
- サービスの仕事: ビジネスロジック（`db: AsyncSession`はパラメータで受け取る）

Claude Codeにこのルールを教えると、自動的にこのパターンでコードを生成します。

---

## 4. 認証の実装（JWT + リフレッシュトークン）

### トークン生成

```python
# core/security.py
import os
from datetime import datetime, timedelta, timezone
from jose import jwt

SECRET_KEY = os.environ.get("JWT_SECRET_KEY", "")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = timedelta(minutes=30)
REFRESH_TOKEN_EXPIRE = timedelta(days=7)


def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + ACCESS_TOKEN_EXPIRE
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + REFRESH_TOKEN_EXPIRE
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

**重要**: `SECRET_KEY`は環境変数から読み込みます。デフォルト値に実際のキーを書かないでください。空文字列をデフォルトにします。

### 認証ミドルウェア

```python
# core/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "access":
            raise HTTPException(status.HTTP_401_UNAUTHORIZED)
        user_id = payload.get("sub")
    except JWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED)

    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED)
    return user
```

---

## 5. テスト戦略（セキュリティテスト必須）

### 全エンドポイントに必須の4テスト

これが最も重要なセクションです。新しいエンドポイントを追加したら、必ず以下の4つのテストを書きます。

```python
# tests/test_workflows_security.py

class TestWorkflowSecurity:
    """すべてのエンドポイントに対するセキュリティテスト"""

    async def test_unauthenticated_returns_401(self, client):
        """認証なし → 401"""
        response = await client.post("/api/v1/workflows")
        assert response.status_code == 401

    async def test_wrong_role_returns_403(self, client, viewer_token):
        """権限不足 → 403"""
        response = await client.post(
            "/api/v1/workflows",
            headers={"Authorization": f"Bearer {viewer_token}"},
            json={"name": "test"},
        )
        assert response.status_code == 403

    async def test_cross_tenant_returns_404(self, client, other_tenant_token):
        """他テナントのリソースへのアクセス → 404（403ではない）"""
        response = await client.get(
            "/api/v1/workflows/existing-id",
            headers={"Authorization": f"Bearer {other_tenant_token}"},
        )
        # 404を返す（403だとリソースの存在がリークする）
        assert response.status_code == 404

    async def test_malformed_input_returns_422(self, client, admin_token):
        """不正な入力 → 422"""
        response = await client.post(
            "/api/v1/workflows",
            headers={"Authorization": f"Bearer {admin_token}"},
            json={"invalid_field": "value"},
        )
        assert response.status_code == 422
```

**なぜ404であって403ではないのか:**

クロステナントアクセスで403を返すと、「そのリソースは存在するが、あなたにはアクセス権がない」という情報がリークします。404なら「そんなリソースは存在しない」としか伝わりません。セキュリティの基本原則です。

### テスト命名規則

```
test_<アクション>_<条件>_<期待結果>

例:
test_create_workflow_unauthenticated_returns_401
test_get_workflow_cross_tenant_returns_404
test_update_workflow_valid_data_returns_200
```

Claude Codeにこの命名規則を教えると、一貫したテストコードを生成してくれます。

---

## 6. ミドルウェアスタックの設計

FastAPIのミドルウェアは「順番」が重要です。

```python
# main.py
app = FastAPI()

# ↓ 外側から順に適用される（リクエストは上→下、レスポンスは下→上）
app.add_middleware(CORSMiddleware, ...)          # 1. CORS
app.add_middleware(SecurityHeadersMiddleware)     # 2. セキュリティヘッダー
app.add_middleware(MetricsMiddleware)             # 3. メトリクス収集
app.add_middleware(CorrelationIdMiddleware)       # 4. リクエストID付与
app.add_middleware(RequestTimingMiddleware)       # 5. 処理時間計測
app.add_middleware(TenantExtractionMiddleware)    # 6. テナント抽出
app.add_middleware(RateLimitMiddleware)           # 7. レート制限
```

**設計原則:**
- セキュリティ系は外側（早期に弾く）
- ビジネスロジック系は内側
- メトリクスはできるだけ外側（全リクエストを計測）
- 各ミドルウェアは100行以下

---

## 7. データベース設計とマイグレーション

### Async SQLAlchemy設定

```python
# core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = os.environ.get("DATABASE_URL", "")

engine = create_async_engine(DATABASE_URL, echo=False)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_db():
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()
```

### Alembicマイグレーション

```bash
# マイグレーション作成
alembic revision --autogenerate -m "add_workflows_table"

# 適用
alembic upgrade head

# ロールバック（必ず可能にしておく）
alembic downgrade -1
```

**鉄則: すべてのマイグレーションは可逆にする。** `down`マイグレーションが書けない変更は、設計を見直してください。

---

## 8. デプロイメント

### ソロプレナー向け推奨スタック

| コンポーネント | サービス | 月額目安 |
|-------------|---------|---------|
| API | Vercel Serverless or Railway | ¥0-2,000 |
| DB | Supabase (PostgreSQL) | ¥0（無料枠） |
| Redis | Upstash | ¥0（無料枠） |
| 監視 | Grafana Cloud | ¥0（無料枠） |

**ポイント**: Kubernetes、自前Docker運用は不要です。マネージドサービスを使えば、インフラ管理コストはゼロに近づきます。

### ヘルスチェックエンドポイント

```python
@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.0.0"}
```

すべてのデプロイサービスにヘルスチェックを設定します。3AM Test: 深夜3時に障害が起きても、自動回復するか優雅にデグレードする設計にしてください。

---

## まとめ

FastAPI × Claude Codeで爆速API開発を実現するポイント：

1. **CLAUDE.mdにプロジェクト構成を書く** — Claude Codeの出力品質が格段に上がる
2. **薄いルーターパターン** — ルーターは300行以下、ビジネスロジックはサービスへ
3. **セキュリティテスト4種は必須** — 401, 403, 404, 422
4. **環境変数で秘密情報管理** — ハードコードは絶対NG
5. **マネージドサービスを使う** — インフラ管理に時間を使わない

Claude Codeにこれらのルールを教えれば、プロダクションレベルのAPIを1時間で構築できます。テンプレートではなく、あなたのプロジェクトに最適化されたコードが手に入ります。

---

*API開発やClaude Code活用について、今後も実践的な情報を発信します。フォローして最新記事をチェックしてください。*