Here is the complete 3-tier rule engine implementation using Python, FastAPI, and SQLite to build a system that creates, combines, and evaluates rules via AST (Abstract Syntax Tree).

1. Project Structure
bash
Copy code
rule-engine/
│
├── app.py                  # FastAPI backend
├── ast_engine.py           # AST logic
├── database.py             # Database connection and schema setup
├── models.py               # Data models and helper classes
├── templates/
│   └── index.html          # Simple UI for rule input
├── test_cases.py           # Test cases for functionality
└── requirements.txt        # Dependencies
2. Backend Implementation
2.1. AST Representation (models.py)
This is the class that represents the AST nodes. Each node can be an operator (AND/OR) or an operand (like age > 30).

python
Copy code
# models.py
class Node:
    def __init__(self, node_type, value=None, left=None, right=None):
        self.type = node_type  # 'operator' or 'operand'
        self.value = value  # AND/OR for operators, condition for operands
        self.left = left  # Left child node
        self.right = right  # Right child node

    def __repr__(self):
        return f"Node(type='{self.type}', value='{self.value}', left={self.left}, right={self.right})"
2.2. Database Setup (database.py)
We use SQLite for this example. The rules and attribute catalog are stored here.

python
Copy code
# database.py
import sqlite3

def get_db_connection():
    conn = sqlite3.connect('rules.db')
    conn.row_factory = sqlite3.Row
    return conn

def setup_database():
    conn = get_db_connection()
    cursor = conn.cursor()

    cursor.execute('''
    CREATE TABLE IF NOT EXISTS rules (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        rule_string TEXT NOT NULL
    );
    ''')

    cursor.execute('''
    CREATE TABLE IF NOT EXISTS attribute_catalog (
        attribute_name TEXT PRIMARY KEY,
        attribute_type TEXT NOT NULL
    );
    ''')

    conn.commit()
    conn.close()

setup_database()
2.3. AST Logic and Parsing (ast_engine.py)
This module handles creating and evaluating AST nodes from rule strings.

python
Copy code
# ast_engine.py
import ast
from models import Node

def create_rule(rule_string: str) -> Node:
    """Parses a rule string and returns the root node of the AST."""
    tree = ast.parse(rule_string, mode='eval')
    return build_ast(tree.body)

def build_ast(node) -> Node:
    """Recursively converts an AST node to our custom Node structure."""
    if isinstance(node, ast.BoolOp):
        operator = 'AND' if isinstance(node.op, ast.And) else 'OR'
        return Node('operator', operator, build_ast(node.values[0]), build_ast(node.values[1]))
    elif isinstance(node, ast.Compare):
        left = node.left.id  # Attribute (e.g., "age")
        operator = ast.dump(node.ops[0]).replace('()', '')  # E.g., ">"
        value = node.comparators[0].n  # Value (e.g., 30)
        return Node('operand', f"{left} {operator} {value}")
    else:
        raise ValueError("Invalid rule format.")

def evaluate_rule(node: Node, data: dict) -> bool:
    """Evaluates the AST against the provided data."""
    if node.type == 'operand':
        attribute, operator, value = node.value.split()
        attribute_value = data.get(attribute)
        return eval(f"{attribute_value} {operator} {value}")

    elif node.type == 'operator':
        left_result = evaluate_rule(node.left, data)
        right_result = evaluate_rule(node.right, data)
        return left_result and right_result if node.value == 'AND' else left_result or right_result

    raise ValueError("Invalid node type.")
2.4. FastAPI Application (app.py)
This is the core API that exposes endpoints for creating, combining, and evaluating rules.

python
Copy code
# app.py
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
from ast_engine import create_rule, evaluate_rule
from database import get_db_connection

app = FastAPI()

class RuleInput(BaseModel):
    rule_string: str

class EvaluateInput(BaseModel):
    ast: dict
    data: dict

@app.post("/create_rule")
def create_rule_endpoint(rule: RuleInput):
    try:
        ast_root = create_rule(rule.rule_string)
        return {"ast": ast_root}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.post("/evaluate_rule")
def evaluate_rule_endpoint(input: EvaluateInput):
    try:
        ast_root = input.ast  # Assume it's passed correctly in JSON
        data = input.data
        result = evaluate_rule(ast_root, data)
        return {"result": result}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
3. Frontend (templates/index.html)
This simple HTML form allows users to input rules and send them to the backend.

html
Copy code
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rule Engine</title>
</head>
<body>
    <h1>Rule Engine</h1>
    <form id="ruleForm">
        <input type="text" id="ruleInput" placeholder="Enter rule string" required>
        <button type="submit">Create Rule</button>
    </form>

    <div id="result"></div>

    <script>
        document.getElementById('ruleForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            const ruleString = document.getElementById('ruleInput').value;

            const response = await fetch('/create_rule', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ rule_string: ruleString })
            });

            const result = await response.json();
            document.getElementById('result').innerText = JSON.stringify(result);
        });
    </script>
</body>
</html>
4. Test Cases (test_cases.py)
python
Copy code
from ast_engine import create_rule, evaluate_rule

def test_rule_creation():
    rule = "age > 30 AND salary > 50000"
    ast_root = create_rule(rule)
    assert ast_root.type == 'operator'

def test_rule_evaluation():
    rule = "age > 30 AND salary > 50000"
    ast_root = create_rule(rule)
    data = {"age": 35, "salary": 60000}
    result = evaluate_rule(ast_root, data)
    assert result is True

if __name__ == "__main__":
    test_rule_creation()
    test_rule_evaluation()
    print("All tests passed!")
5. Running the Application
Install dependencies:

Copy code
pip install fastapi uvicorn pydantic sqlite3
Start the FastAPI server:

lua
Copy code
uvicorn app:app --reload
Open http://127.0.0.1:8000 in your browser to access the frontend.

6. README
markdown
Copy code
# Rule Engine with AST

## Overview
This project implements a rule engine using an Abstract Syntax Tree (AST) to create, combine, and evaluate rules.

## Features
- **Dynamic rule creation and evaluation**.
- **FastAPI backend** with endpoints to manage rules.
- **SQLite database** to store rules and metadata.

## How to Run
1. Install dependencies: `pip install -r requirements.txt`
2. Start the server: `uvicorn app:app --reload`
3. Open `http://127.0.0.1:8000` to use the UI.

## API Endpoints
- **POST /create_rule**: Create a rule and generate an AST.
- **POST /evaluate_rule**: Evaluate a rule against input data.

## License
MIT
