## Steps to run the ptoject
Follow the steps to setup and test the identify point locally
### Clone the repository 
bash
git clone https://github.com/SHAIKRIYAZZZZ/moonrider-backend-assessment.git
cd moonrider-backend-assessment
## Install dependies
pip install -r requirements.txt

## Run the Flask Server
python.app.py

## Use curl to test the API
http://localhost:5000/identify

# moonrider-backend-assesment
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from utils import merge_contact_data

app = Flask(_moon_rider_)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Define the Contact model
class Contact(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    phoneNumber = db.Column(db.String(20), nullable=True)
    email = db.Column(db.String(120), nullable=True)
    linkedId = db.Column(db.Integer, nullable=True)  # points to primary contact
    linkPrecedence = db.Column(db.String(10), default='primary')  # 'primary' or 'secondary'
    createdAt = db.Column(db.DateTime, default=datetime.utcnow)
    updatedAt = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    deletedAt = db.Column(db.DateTime, nullable=True)

@app.before_first_request
def create_tables():
    db.create_all()

@app.route('/identify', methods=['POST'])
def identify():
    data = request.get_json()
    email = data.get('email')
    phone = data.get('phoneNumber')

    if not email and not phone:
        return jsonify({"error": "At least one of email or phoneNumber is required"}), 400

    matches = Contact.query.filter(
        (Contact.email == email) | (Contact.phoneNumber == phone)
    ).all()

    if not matches:
        # No match - create new primary contact
        new_contact = Contact(email=email, phoneNumber=phone, linkPrecedence="primary")
        db.session.add(new_contact)
        db.session.commit()
        return jsonify(merge_contact_data(new_contact, [])), 200

    # Determine primary
    primary = next((c for c in matches if c.linkPrecedence == "primary"), matches[0])

    # Check if new information is provided
    already_exists = any(
        (c.email == email and c.phoneNumber == phone)
        for c in matches
    )

    if not already_exists:
        new_secondary = Contact(
            email=email,
            phoneNumber=phone,
            linkPrecedence="secondary",
            linkedId=primary.id
        )
        db.session.add(new_secondary)
        db.session.commit()
        matches.append(new_secondary)

    return jsonify(merge_contact_data(primary, matches)), 200

if _name_ == '_main_':
    app.run(debug=True)
