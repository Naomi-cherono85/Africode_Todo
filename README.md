# Flask To-Do List Application

This is a simple To-Do list application built with Flask, SQLAlchemy, and Flask-WTF.

## Features

- Add new to-do items with a description and category.
- Update the status of to-do items (mark as completed).
- Delete to-do items.
- Filter to-do items by category.

## Prerequisites

Before you begin, ensure you have the following installed on your local machine:

- Python 3.7 or higher
- PostgreSQL

## Installation

1. **Clone the repository:**

    ```sh
    git clone https://github.com/your-username/todo-flask-app.git
    cd todo-flask-app
    ```

2. **Create and activate a virtual environment:**

    ```sh
    python3 -m venv venv
    source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
    ```

3. **Install the required dependencies:**

    ```sh
    pip install -r requirements.txt
    ```

4. **Configure the Database:**

    - Make sure PostgreSQL is running on your machine.
    - Create a database named `todoapp` in PostgreSQL.
    - Update the database URI in the Flask application configuration:

    ```python
    app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://<username>:<password>@localhost:5432/todoapp'
    ```

5. **Run database migrations:**

    ```sh
    flask db upgrade
    ```

## Usage

1. **Run the Flask application:**

    ```sh
    python app.py
    ```

2. **Open your browser and navigate to:**

    ```
    http://127.0.0.1:8000
    ```

## Application Structure

- **`app.py`**: The main Flask application file.
- **`templates/index.html`**: The HTML template for the home page.
- **`requirements.txt`**: The list of Python packages required for the application.

## Code Overview

### `app.py`

```python
from flask import Flask, render_template, request, redirect, url_for, flash
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField, SelectField
from wtforms.validators import DataRequired
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy.exc import SQLAlchemyError
from flask_migrate import Migrate
import secrets

SECRET_KEY = secrets.token_hex(16)

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://<username>:<password>@localhost:5432/todoapp'
app.config['SECRET_KEY'] = SECRET_KEY
db = SQLAlchemy(app)
migrate = Migrate(app, db)

class Todo(db.Model):
    __tablename__ = 'todos'
    id = db.Column(db.Integer, primary_key=True)
    description = db.Column(db.String(20), nullable=False)
    completed = db.Column(db.Boolean, nullable=False, default=False)
    category_id = db.Column(db.Integer, db.ForeignKey('categories.id'))
    category = db.relationship('Category', back_populates='todos')

    def __repr__(self):
        return f'{self.id} {self.description}'

class Category(db.Model):
    __tablename__ = 'categories'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(20), nullable=False)
    todos = db.relationship('Todo', back_populates='category')

    def __repr__(self):
        return f'{self.id} {self.name}'

class TodoForm(FlaskForm):
    description = StringField('Description', validators=[DataRequired()])
    category_id = SelectField('Category', validators=[DataRequired()])
    submit = SubmitField('Submit')

    def __init__(self, *args, **kwargs):
        super(TodoForm, self).__init__(*args, **kwargs)
        self.category_id.choices = [(category.id, category.name) for category in Category.query.order_by(Category.name).all()]

@app.route('/update-todo_status/<int:todo_id>', methods=['POST'])
def update_todo_status(todo_id):
    try:
        todo = Todo.query.get_or_404(todo_id)
        todo.completed = 'completed' in request.form
        db.session.commit()
        return redirect(url_for('index'))
    except SQLAlchemyError as e:
        db.session.rollback()
        flash(f'Failed to update todo status: {str(e)}', 'danger')
        return redirect(url_for('index'))

@app.route('/delete-todo_status/<int:todo_id>', methods=['POST'])
def delete_todo(todo_id):
    try:
        todo = Todo.query.get_or_404(todo_id)
        db.session.delete(todo)
        db.session.commit()
        flash(f'{todo.description} deleted successfully', 'success')
        return redirect(url_for('index'))
    except SQLAlchemyError as e:
        db.session.rollback()
        flash(f'Failed to delete todo: {str(e)}', 'danger')
        return redirect(url_for('index'))

@app.route('/', methods=['GET', 'POST'])
@app.route('/category/<int:category_id>', methods=['GET'])
def index(category_id=None):
    form = TodoForm()
    if form.validate_on_submit():
        try:
            todo = Todo(description=form.description.data, category_id=form.category_id.data)
            db.session.add(todo)
            db.session.commit()
            flash('Todo added successfully', 'success')
        except SQLAlchemyError as e:
            db.session.rollback()
            flash(f'Failed to add todo item: {str(e)}', 'danger')
        return redirect(url_for('index'))
    if category_id:
        todos = Todo.query.filter_by(category_id=category_id).all()
        current_category = Category.query.get_or_404(category_id)
    else:
        todos = Todo.query.order_by(Todo.id.desc()).all()
        current_category = None
    categories = Category.query.all()
    return render_template('index.html', todos=todos, form=form, current_category=current_category, categories=categories)

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8000, debug=True)
