from flask import Flask, render_template, request, redirect, url_for, jsonify
import os
import json
import csv
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)

# Configuración base de datos SQLite
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database/usuarios.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Modelo para la tabla usuarios
class Usuario(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), nullable=False)

    def __repr__(self):
        return f'<Usuario {self.nombre}>'

# Crear base de datos y tablas si no existen
with app.app_context():
    db.create_all()

# Rutas básicas
@app.route('/')
def index():
    return render_template('index.html')

@app.route('/formulario')
def formulario():
    return render_template('formulario.html')

# Ruta para procesar formulario y guardar datos en TXT, JSON, CSV y SQLite
@app.route('/guardar', methods=['POST'])
def guardar():
    nombre = request.form['nombre']
    email = request.form['email']

    # Guardar en archivo TXT
    with open('datos/datos.txt', 'a', encoding='utf-8') as f:
        f.write(f'{nombre},{email}\n')

    # Guardar en archivo JSON
    datos_json = []
    json_path = 'datos/datos.json'
    if os.path.exists(json_path):
        with open(json_path, 'r', encoding='utf-8') as f:
            try:
                datos_json = json.load(f)
            except json.JSONDecodeError:
                datos_json = []
    datos_json.append({'nombre': nombre, 'email': email})
    with open(json_path, 'w', encoding='utf-8') as f:
        json.dump(datos_json, f, indent=4)

    # Guardar en archivo CSV
    csv_path = 'datos/datos.csv'
    file_exists = os.path.isfile(csv_path)
    with open(csv_path, 'a', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(['nombre', 'email'])  # encabezados
        writer.writerow([nombre, email])

    # Guardar en base de datos SQLite
    nuevo_usuario = Usuario(nombre=nombre, email=email)
    db.session.add(nuevo_usuario)
    db.session.commit()

    return redirect(url_for('resultado'))

# Ruta para mostrar resultado simple
@app.route('/resultado')
def resultado():
    return render_template('resultado.html')

# Rutas para leer datos desde archivos y base de datos

@app.route('/leer_txt')
def leer_txt():
    datos = []
    try:
        with open('datos/datos.txt', 'r', encoding='utf-8') as f:
            for linea in f:
                nombre, email = linea.strip().split(',')
                datos.append({'nombre': nombre, 'email': email})
    except FileNotFoundError:
        datos = []
    return jsonify(datos)

@app.route('/leer_json')
def leer_json():
    try:
        with open('datos/datos.json', 'r', encoding='utf-8') as f:
            datos = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        datos = []
    return jsonify(datos)

@app.route('/leer_csv')
def leer_csv():
    datos = []
    try:
        with open('datos/datos.csv', 'r', encoding='utf-8') as f:
            reader = csv.DictReader(f)
            for row in reader:
                datos.append(row)
    except FileNotFoundError:
        datos = []
    return jsonify(datos)

@app.route('/leer_db')
def leer_db():
    usuarios = Usuario.query.all()
    datos = [{'nombre': u.nombre, 'email': u.email} for u in usuarios]
    return jsonify(datos)

if __name__ == '__main__':
    app.run(debug=True)
