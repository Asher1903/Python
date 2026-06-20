# In-Depth Guide: Python Type Hints & Pydantic Validation

This guide explains how **Type Hints** and **Pydantic** work together to provide clean, self-documenting, and validated APIs in FastAPI.

---

## 1. What are Python Type Hints?

Historically, Python is a dynamically-typed language. You declare variables without defining what data type they should hold. Type hints (introduced in Python 3.5 via PEP 484) allow you to specify the expected types of variables, function parameters, and return types.

### Syntax Comparison

Without Type Hints (Traditional Python):
```python
def calculate_price(price, discount):
    return price - discount
```
*Problem:* It is not clear whether `price` and `discount` should be integers, floats, or decimal objects. If someone passes a string `"10"`, the code will crash at runtime with a `TypeError`.

With Type Hints:
```python
def calculate_price(price: float, discount: float) -> float:
    return price - discount
```
*   `price: float` and `discount: float` specify that the inputs must be floating-point numbers.
*   `-> float` specifies that the function will return a float.

> [!NOTE]
> Standard Python **ignores type hints at runtime**. They are only used by external tools like static checkers (`mypy`) or your code editor to provide autocomplete and highlights. **Pydantic changes this by enforcing them at runtime.**

---

## 2. What is Pydantic?

**Pydantic** is a data validation and parsing library built on Python type hints. When you define a schema in Pydantic, the library checks data types at runtime and guarantees that code inside your application matches those types.

### The `BaseModel`

To create a Pydantic schema, you define a class that inherits from `pydantic.BaseModel`:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    is_active: bool = True  # Default value
```

When you initialize this class with data, Pydantic parses it:
```python
# Valid Data
user = User(id=12, name="Asher")
print(user.id)        # 12
print(user.is_active) # True (default value used)
```

---

## 3. Data Parsing vs. Validation (Type Coercion)

Pydantic is a **parser**, not just a validator. This means that if it receives data that doesn't match the type hint exactly but can be safely converted, it will automatically cast (coerce) it.

```python
# Notice: 'id' is passed as a string, and 'is_active' is passed as an integer
raw_data = {
    "id": "123",
    "name": "Sarah",
    "is_active": 1
}

user = User(**raw_data)

print(type(user.id))        # <class 'int'> (Casted from string "123" to integer 123)
print(type(user.is_active)) # <class 'bool'> (Casted from integer 1 to boolean True)
```

If Pydantic cannot convert a value (e.g. trying to cast `"hello"` to an integer), it raises a `ValidationError`.

---

## 4. Advanced Field Constraints

You can use the `Field` class from Pydantic to enforce specific validation rules on data attributes:

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional

class Product(BaseModel):
    id: int
    # String must be at least 3 characters and at most 100 characters
    title: str = Field(min_length=3, max_length=100)
    # Price must be greater than zero
    price: float = Field(gt=0)
    # Description is optional and defaults to None
    description: Optional[str] = None
```

### Common `Field` Constraints:
*   `gt`: Greater than
*   `ge`: Greater than or equal to
*   `lt`: Less than
*   `le`: Less than or equal to
*   `min_length` / `max_length`: String size boundaries
*   `pattern`: RegEx pattern matching (e.g., matching phone number formats)

---

## 5. Complete Runnable Code Example

Here is a full code example displaying how to parse data, capture validation errors, and serialize data back to JSON.

```python
from pydantic import BaseModel, Field, EmailStr, ValidationError
from typing import Optional

# 1. Define the Schema
class UserRegister(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(gt=0, lt=130)
    website: Optional[str] = None

# 2. Test parsing valid data
valid_input = {
    "username": "coder123",
    "email": "coder@python.org",
    "age": "25"  # String will be coerced to integer
}

print("--- Parsing Valid Data ---")
try:
    user = UserRegister(**valid_input)
    print(f"Successfully validated: {user}")
    print(f"Age type: {type(user.age)}")
except ValidationError as e:
    print(e.json())

# 3. Test parsing invalid data (Validation Fails)
invalid_input = {
    "username": "a",             # Too short (min_length=3)
    "email": "not-a-valid-email", # Invalid email syntax
    "age": 150                   # Too old (lt=130)
}

print("\n--- Parsing Invalid Data ---")
try:
    bad_user = UserRegister(**invalid_input)
except ValidationError as e:
    print("Validation failed with the following errors:")
    # Print the errors in a clean JSON format
    print(e.json(indent=2))
```

---

## 6. How FastAPI uses Pydantic in HTTP Requests

In FastAPI, when you declare a parameter in your endpoint function with a Pydantic model type hint, FastAPI handles all the web boilerplate:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create_item(item: Item):
    # 1. FastAPI reads the incoming HTTP JSON request body
    # 2. It parses it through Item(name=..., price=...)
    # 3. If validation succeeds, 'item' is injected into this function
    # 4. If validation fails, FastAPI returns a 422 HTTP error response automatically.
    return {"message": f"Created item {item.name} costing ${item.price}"}
```
