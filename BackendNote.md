# Backend Quick Notes

Quick reference for a **FastAPI + PostgreSQL + React** recipe app.

## 1. Python Environment (`uv`)

```bash
python -m pip install uv
uv python pin 3.12.7
uv venv
uv pip install -r requirements.txt
# Or add packages to the project:
uv add "fastapi[standard]" "uvicorn[standard]"
uv run fastapi dev main.py
```

FastAPI Swagger docs: `http://localhost:8000/docs`

## 2. REST API Cheatsheet

| Method | Endpoint | Use |
|---|---|---|
| `GET` | `/recipes` | List recipes |
| `POST` | `/recipes` | Create a recipe |
| `PUT` | `/recipes/{recipe_id}` | Update a recipe |
| `DELETE` | `/recipes/{recipe_id}` | Delete a recipe |

Use **plural nouns** for resource collections; the HTTP method defines the action.

## 3. Project Structure

| File | Responsibility |
|---|---|
| `main.py` | API routes and dependency injection |
| `schemas.py` | Validate requests and shape responses with Pydantic |
| `models.py` | SQLAlchemy database table definitions |
| `database.py` | Engine, connection pool and session factory |
| `.env` | Environment variables and credentials |
| `requirements.txt` | Python dependencies |
| `.venv/` | Project virtual environment |

- **SQLAlchemy** maps Python classes to database tables.
- **psycopg** lets Python connect to PostgreSQL.
- **Pydantic** validates API data and helps generate documentation.

## 4. Database Connection

In `.env`:

```env
DATABASE_URL=postgresql+psycopg://postgres:yourpassword@localhost:5432/recipe_db
```

Replace the example credentials with your own; do not commit secrets.

## 5. SQLAlchemy Model (`models.py`)

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

## 6. Pydantic Schemas (`schemas.py`)

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

**Remember:** `models.py` defines database columns; `schemas.py` defines validated API input/output.

## 7. FastAPI Endpoint Examples (`main.py`)

Create a recipe:

```python
@app.post("/recipes/", response_model=schemas.RecipeOut, status_code=status.HTTP_201_CREATED)
async def create_recipe(recipe: schemas.RecipeBase, db: AsyncSession = Depends(get_db)):
    new_recipe = Recipe(**recipe.model_dump())
    db.add(new_recipe)
    await db.commit()
    await db.refresh(new_recipe)
    return new_recipe
```

Search by ingredient and maximum cooking time:

```python
@app.get("/recipes/search", response_model=list[schemas.RecipeOut])
async def search_recipes(ingredient: str, max_cooking_time: int, db: AsyncSession = Depends(get_db)):
    query = select(Recipe).where(
        Recipe.ingredients.icontains(ingredient, autoescape=True),
        Recipe.cooking_time <= max_cooking_time,
    )
    return (await db.scalars(query)).all()
```

## 8. Enable CORS (React → FastAPI)

Add middleware **after creating the FastAPI app and before defining routes**. Allow your frontend origin explicitly.

```python
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(lifespan=lifespan)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Change the origin to match your frontend URL. Avoid `*` for production when credentials are enabled.

## 9. React + Axios

Install:

```bash
npm install axios
```

Basic setup and requests:

```jsx
import axios from 'axios';

const API_URL = 'http://localhost:8000/recipes/';

const fetchRecipes = async () => {
  const { data } = await axios.get(API_URL);
  setRecipes(data);
};

const createRecipe = async (formData) => {
  await axios.post(API_URL, {
    ...formData,
    cooking_time: Number(formData.cooking_time),
  });
  await fetchRecipes();
};

const updateRecipe = async (id, formData) => {
  await axios.put(`${API_URL}${id}`, {
    ...formData,
    cooking_time: Number(formData.cooking_time),
  });
  await fetchRecipes();
};

const deleteRecipe = async (id) => {
  await axios.delete(`${API_URL}${id}`);
  await fetchRecipes();
};
```

Useful React patterns:

```jsx
const [recipes, setRecipes] = useState([]);
const [error, setError] = useState(null);
const [editingId, setEditingId] = useState(null);

useEffect(() => {
  fetchRecipes().catch(() => setError('Failed to load recipes.'));
}, []);

const handleChange = (e) => {
  setFormData({ ...formData, [e.target.name]: e.target.value });
};
```

- Use `useEffect` to load data when the component mounts.
- Convert numeric form values, such as `cooking_time`, with `Number(...)` before sending.
- Catch request errors and show a useful message in the UI.
- Use `editingId` to track which recipe is being edited; confirm before deleting.
