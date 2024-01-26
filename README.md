# Farmwiseai-backend-
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager, jwt_required, create_access_token, get_jwt_identity

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///books.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JWT_SECRET_KEY'] = 'your_jwt_secret_key'

db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)

class Book(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    author = db.Column(db.String(50), nullable=False)
    isbn = db.Column(db.String(13), unique=True, nullable=False)
    price = db.Column(db.Float, nullable=False)
    quantity = db.Column(db.Integer, nullable=False)

# Routes

@app.route('/api/books', methods=['GET'])
@jwt_required()
def get_all_books():
    books = Book.query.all()
    book_list = []
    for book in books:
        book_list.append({
            'id': book.id,
            'title': book.title,
            'author': book.author,
            'isbn': book.isbn,
            'price': book.price,
            'quantity': book.quantity
        })
    return jsonify({'books': book_list})

@app.route('/api/books/<isbn>', methods=['GET'])
@jwt_required()
def get_book(isbn):
    book = Book.query.filter_by(isbn=isbn).first()
    if book:
        return jsonify({
            'id': book.id,
            'title': book.title,
            'author': book.author,
            'isbn': book.isbn,
            'price': book.price,
            'quantity': book.quantity
        })
    return jsonify({'message': 'Book not found'}), 404

@app.route('/api/books', methods=['POST'])
@jwt_required()
def add_book():
    data = request.get_json()
    new_book = Book(
        title=data['title'],
        author=data['author'],
        isbn=data['isbn'],
        price=data['price'],
        quantity=data['quantity']
    )
    db.session.add(new_book)
    db.session.commit()
    return jsonify({'message': 'Book added successfully'}), 201

@app.route('/api/books/<isbn>', methods=['PUT'])
@jwt_required()
def update_book(isbn):
    book = Book.query.filter_by(isbn=isbn).first()
    if book:
        data = request.get_json()
        book.title = data['title']
        book.author = data['author']
        book.price = data['price']
        book.quantity = data['quantity']
        db.session.commit()
        return jsonify({'message': 'Book updated successfully'})
    return jsonify({'message': 'Book not found'}), 404

@app.route('/api/books/<isbn>', methods=['DELETE'])
@jwt_required()
def delete_book(isbn):
    book = Book.query.filter_by(isbn=isbn).first()
    if book:
        db.session.delete(book)
        db.session.commit()
        return jsonify({'message': 'Book deleted successfully'})
    return jsonify({'message': 'Book not found'}), 404

# Authentication

@app.route('/api/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    # You would typically fetch the user from the database based on the username
    # For simplicity, let's assume there is a user with username 'admin' and password 'password'
    if username == 'admin' and bcrypt.check_password_hash('$2b$12$2O/B1OGTje.juSL8b/ev8OyRgVJ9q4RU2d.YgKkbPMyu4RqRu6oh6', password):
        access_token = create_access_token(identity=username)
        return jsonify(access_token=access_token)
    else:
        return jsonify({'message': 'Invalid credentials'}), 401

# Token validation decorator
@jwt.token_in_blacklist_loader
def check_if_token_in_blacklist(decrypted_token):
    # This function is called to check if a token has been revoked.
    # You can implement token revocation logic here.
    return False

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
