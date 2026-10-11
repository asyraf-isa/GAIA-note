# Backend Development Notes

Quick reference for building a FastAPI backend with `uv`, Pydantic, SQLAlchemy, and PostgreSQL.

## 1. Python Environment with `uv`

Install `uv`:

```bash
python -m pip install uv
```

Pin the Python version:

```bash
uv python pin 3.12.7
```

Create a virtual environment:

```bash
uv venv
```

Install dependencies from `requirements.txt`:

```bash
uv pip install -r requirements.txt
```

Or add packages directly:

```bash
uv add "fastapi[standard]" "uvicorn[standard]"
```

## 2. Run FastAPI

Start the development server:

```bash
uv run fastapi dev main.py
```

FastAPI's interactive API documentation is available at:

```text
http://localhost:8000/docs
```

## 3. RESTful API Design

Use plural nouns for resource collections. The HTTP method defines the action.

| HTTP method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/recipes` | Create a recipe |
| `GET` | `/recipes` | Retrieve recipes |
| `PUT` | `/recipes/{recipe_id}` | Update a recipe |
| `DELETE` | `/recipes/{recipe_id}` | Delete a recipe |

## 4. Pydantic Schemas — `schemas.py`

Schemas validate request data and serialize responses. FastAPI also uses them to generate API documentation.

```python
from typing import Optional

from pydantic import BaseModel, ConfigDict, Field


class RecipeBase(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    ingredients: str = Field(min_length=1)
    instructions: str = Field(min_length=10)
    cooking_time: int = Field(gt=0)
    image: Optional[str] = Field(default=None, max_length=255)


class RecipeOut(RecipeBase):
    id: int

    model_config = ConfigDict(from_attributes=True)


class RecipeUpdate(BaseModel):
    title: Optional[str] = Field(default=None, min_length=1, max_length=100)
    ingredients: Optional[str] = Field(default=None, min_length=1)
    instructions: Optional[str] = Field(default=None, min_length=10)
    cooking_time: Optional[int] = Field(default=None, gt=0)
    image: Optional[str] = Field(default=None, max_length=255)
```

## 5. Example API Endpoints — `main.py`

### Create a recipe

```python
@app.post(
    "/recipes/",
    response_model=schemas.RecipeOut,
    status_code=status.HTTP_201_CREATED,
)
async def create_recipe(
    recipe: schemas.RecipeBase,
    db: AsyncSession = Depends(get_db),
):
    new_recipe = Recipe(**recipe.model_dump())
    db.add(new_recipe)
    await db.commit()
    await db.refresh(new_recipe)
    return new_recipe
```

### Search recipes by ingredient and maximum cooking time

```python
@app.get("/recipes/search", response_model=list[schemas.RecipeOut])
async def search_recipes(
    ingredient: str,
    max_cooking_time: int,
    db: AsyncSession = Depends(get_db),
):
    query = select(Recipe).where(
        Recipe.ingredients.icontains(ingredient, autoescape=True),
        Recipe.cooking_time <= max_cooking_time,
    )
    return (await db.scalars(query)).all()
```

## 6. Example Project Structure

Separating responsibilities across files makes a project easier to maintain and scale.

| File or directory | Responsibility |
|---|---|
| `main.py` | Entry point, routes, lifespan logic, and dependency injection |
| `schemas.py` | Pydantic models for validation and API serialization |
| `models.py` | SQLAlchemy database table definitions |
| `database.py` | Engine configuration, connection pooling, and session factory |
| `.environment` | Environment variables and credentials *(as noted in the original notes)* |
| `requirements.txt` | Package dependencies |
| `.venv/` | Project-specific virtual environment |

## 7. Database Tools

**Psycopg** is a Python database adapter that allows an application to connect to PostgreSQL, execute SQL, and work with query results.

**SQLAlchemy** is a Python SQL toolkit and Object Relational Mapper (ORM). It lets you work with database records through Python objects while retaining the flexibility of SQL.

## 8. Environment Configuration — `.env`

Example PostgreSQL connection string:

```dotenv
DATABASE_URL=postgresql+psycopg://postgres:yourpassword@localhost:5432/recipe_db
```

## 9. SQLAlchemy Models — `models.py`

```python
from typing import Optional

from sqlalchemy import Integer, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Recipe(Base):
    __tablename__ = "recipes"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    ingredients: Mapped[str] = mapped_column(Text)
    instructions: Mapped[str] = mapped_column(Text)
    cooking_time: Mapped[int] = mapped_column(Integer)
    image: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
```

## 10. `models.py` vs. `schemas.py`

| Responsibility | `models.py` — SQLAlchemy | `schemas.py` — Pydantic |
|---|---|---|
| Works with | PostgreSQL | HTTP clients / API data |
| Defines | Database table columns | Request and response data shapes |
| Enforces | Database constraints, such as column types and nullability | Input validation, such as `min_length` and `gt=0` |
| Typical classes | Database entity or entities | `Base`, response, and update schemas for different use cases |
