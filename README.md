# Recipe Generator

An AI-powered web application that generates recipes based on user-provided ingredients.

## Features

- Input ingredients to receive a unique recipe.
- Download the generated recipe as a PDF.
- View history of generated recipes.

## Technologies Used

- Python
- Flask
- OpenAI API
- SQLite
- HTML/CSS

## Setup Instructions

1. Clone the repository:
   ```bash
   git clone https://github.com/faslan34/Recipe_generator.git
   cd Recipe_generator

 # REcipe generator Code

from flask import Flask, render_template, request, redirect, url_for, make_response, render_template_string
from flask_sqlalchemy import SQLAlchemy
import openai
import os
from dotenv import load_dotenv
from datetime import datetime
from xhtml2pdf import pisa
from io import BytesIO

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")

app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///recipes.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
db = SQLAlchemy(app)

# Database model
class RecipeHistory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    ingredients = db.Column(db.Text, nullable=False)
    recipe_text = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

# Home page: Generate recipe
@app.route("/", methods=["GET", "POST"])
def index():
    recipe = ""
    ingredients_list = []

    if request.method == "POST":
        raw_ingredients = request.form["ingredients"]
        ingredients_list = [i.strip() for i in raw_ingredients.split(",") if i.strip()]

        prompt = f"Create a delicious recipe using the following ingredients: {', '.join(ingredients_list)}. Include ingredients list and preparation steps."

        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "You are a helpful and creative chef."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=500
        )

        recipe = response["choices"][0]["message"]["content"]

        # Save to database
        new_entry = RecipeHistory(ingredients=", ".join(ingredients_list), recipe_text=recipe)
        db.session.add(new_entry)
        db.session.commit()

    return render_template("index.html", recipe=recipe, ingredients=ingredients_list)

# View history of recipes
@app.route("/history")
def history():
    all_recipes = RecipeHistory.query.order_by(RecipeHistory.created_at.desc()).all()
    return render_template("history.html", recipes=all_recipes)

# View individual recipe
@app.route("/recipe/<int:recipe_id>")
def view_recipe(recipe_id):
    recipe = RecipeHistory.query.get_or_404(recipe_id)
    ingredients = recipe.ingredients.split(",")
    return render_template("recipe.html", recipe=recipe, ingredients=ingredients)

# Download as PDF
@app.route("/download-pdf", methods=["POST"])
def download_pdf():
    ingredients = request.form["ingredients"].split(",")
    recipe = request.form["recipe"]

    pdf_template = """
    <html>
    <head><meta charset="UTF-8"></head>
    <body>
        <h1>AI-Generated Recipe</h1>
        <h2>Ingredients:</h2>
        <ul>
        {% for item in ingredients %}
            <li>{{ item }}</li>
        {% endfor %}
        </ul>
        <h2>Recipe:</h2>
        <pre>{{ recipe }}</pre>
    </body>
    </html>
    """

    html = render_template_string(pdf_template, ingredients=ingredients, recipe=recipe)

    result = BytesIO()
    pisa_status = pisa.CreatePDF(html, dest=result)

    if pisa_status.err:
        return "PDF generation failed", 500

    response = make_response(result.getvalue())
    response.headers["Content-Type"] = "application/pdf"
    response.headers["Content-Disposition"] = "attachment; filename=recipe.pdf"

    return response

if __name__ == "__main__":
    with app.app_context():
        db.create_all()
    app.run(debug=True)
