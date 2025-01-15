from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager, create_access_token, jwt_required

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///hospital.db'
app.config['JWT_SECRET_KEY'] = 'your_secret_key'
db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)

# Database Models
class Doctor(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(200))

class Patient(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    ward = db.Column(db.String(100))
    medical_record = db.Column(db.Text)
    doctor_id = db.Column(db.Integer, db.ForeignKey('doctor.id'))

# Initialize Database
with app.app_context():
    db.create_all()

# Register Doctor
@app.route('/register', methods=['POST'])
def register():
    data = request.json
    hashed_password = bcrypt.generate_password_hash(data['password']).decode('utf-8')
    new_doctor = Doctor(name=data['name'], email=data['email'], password=hashed_password)
    db.session.add(new_doctor)
    db.session.commit()
    return jsonify({"message": "Doctor registered successfully!"}), 201

# Login
@app.route('/login', methods=['POST'])
def login():
    data = request.json
    doctor = Doctor.query.filter_by(email=data['email']).first()
    if doctor and bcrypt.check_password_hash(doctor.password, data['password']):
        access_token = create_access_token(identity=doctor.id)
        return jsonify({"access_token": access_token}), 200
    return jsonify({"message": "Invalid credentials!"}), 401

# Get Patients for Doctor
@app.route('/patients', methods=['GET'])
@jwt_required()
def get_patients():
    doctor_id = get_jwt_identity()
    patients = Patient.query.filter_by(doctor_id=doctor_id).all()
    return jsonify([{
        "id": patient.id,
        "name": patient.name,
        "ward": patient.ward,
        "medical_record": patient.medical_record
    }])

if __name__ == '__main__':
    app.run(debug=True)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Doctor Login</title>
</head>
<body>
    <h1>Login</h1>
    <form id="loginForm">
        <label>Email:</label><br>
        <input type="email" id="email" required><br>
        <label>Password:</label><br>
        <input type="password" id="password" required><br><br>
        <button type="submit">Login</button>
    </form>
    <script>
        document.getElementById('loginForm').addEventListener('submit', async (event) => {
            event.preventDefault();
            const email = document.getElementById('email').value;
            const password = document.getElementById('password').value;

            const response = await fetch('http://127.0.0.1:5000/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ email, password })
            });

            const data = await response.json();
            if (response.status === 200) {
                alert('Login Successful!');
                console.log('Token:', data.access_token);
            } else {
                alert(data.message);
            }
        });
    </script>
</body>
</html>
