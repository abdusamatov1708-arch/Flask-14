# Flask-14
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    notes = db.relationship('Note', backref='owner', lazy=True, cascade='all, delete-orphan')

    def to_dict(self):
        return {'id': self.id, 'username': self.username}

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    def to_dict(self):
        return {
            'id': self.id,
            'title': self.title,
            'body': self.body,
            'user_id': self.user_id,
            'created_at': self.created_at.isoformat(),
        }


from flask import Blueprint, request, jsonify, session
from app.models import db, User

auth_bp = Blueprint('auth', __name__)


@auth_bp.route('/login', methods=['POST'])
def login():
    data = request.get_json(silent=True) or {}
    username = (data.get('username') or '').strip()

    if len(username) < 2:
        return jsonify({'error': 'username kamida 2 ta belgi bo\'lishi shart'}), 400

    user = User.query.filter_by(username=username).first()
    if not user:
        user = User(username=username)
        db.session.add(user)
        db.session.commit()

    session['user_id'] = user.id
    return jsonify({'user': user.to_dict(), 'message': 'Login muvaffaqiyatli'}), 200


@auth_bp.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return '', 204


@auth_bp.route('/me', methods=['GET'])
def me():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': 'Login qilinmagan'}), 401
    user = User.query.get(user_id)
    if not user:
        return jsonify({'error': 'Foydalanuvchi topilmadi'}), 404
    return jsonify({'user': user.to_dict()}), 200


from flask import Blueprint, request, jsonify, session
from app.models import db, Note, User

notes_bp = Blueprint('notes', __name__)


def get_current_user():
    user_id = session.get('user_id')
    if not user_id:
        return None
    return User.query.get(user_id)


@notes_bp.route('', methods=['GET'])
def list_notes():
    user = get_current_user()
    if not user:
        return jsonify({'error': 'Avval tizimga kiring'}), 401

    notes = Note.query.filter_by(user_id=user.id).all()
    return jsonify({'notes': [n.to_dict() for n in notes]}), 200


@notes_bp.route('', methods=['POST'])
def create_note():
    user = get_current_user()
    if not user:
        return jsonify({'error': 'Avval tizimga kiring'}), 401

    data = request.get_json(silent=True) or {}
    title = (data.get('title') or '').strip()
    body = (data.get('body') or '').strip()

    if not title or not body:
        return jsonify({'error': 'title va body to\'ldirilishi shart'}), 400

    note = Note(title=title, body=body, user_id=user.id)
    db.session.add(note)
    db.session.commit()
    return jsonify({'note': note.to_dict()}), 201


@notes_bp.route('/<int:id>', methods=['GET'])
def get_note(id):
    user = get_current_user()
    if not user:
        return jsonify({'error': 'Avval tizimga kiring'}), 401

    note = Note.query.get_or_404(id)
    if note.user_id != user.id:
        return jsonify({'error': 'Ruxsat yo\'q'}), 403

    return jsonify({'note': note.to_dict()}), 200


@notes_bp.route('/<int:id>', methods=['PUT'])
def update_note(id):
    user = get_current_user()
    if not user:
        return jsonify({'error': 'Avval tizimga kiring'}), 401

    note = Note.query.get_or_404(id)
    if note.user_id != user.id:
        return jsonify({'error': 'Ruxsat yo\'q'}), 403

    data = request.get_json(silent=True) or {}
    if 'title' in data:
        note.title = (data['title'] or '').strip()
    if 'body' in data:
        note.body = (data['body'] or '').strip()

    db.session.commit()
    return jsonify({'note': note.to_dict()}), 200


@notes_bp.route('/<int:id>', methods=['DELETE'])
def delete_note(id):
    user = get_current_user()
    if not user:
        return jsonify({'error': 'Avval tizimga kiring'}), 401

    note = Note.query.get_or_404(id)
    if note.user_id != user.id:
        return jsonify({'error': 'Ruxsat yo\'q'}), 403

    db.session.delete(note)
    db.session.commit()
    return '', 204


from flask import Flask, jsonify
from flask_cors import CORS
from app.models import db
from app.auth import auth_bp
from app.notes import notes_bp


def create_app():
    app = Flask(__name__)
    app.secret_key = 'juda-maxfiy-kalit-bu-yerda'
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes_api.db'
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

    db.init_app(app)
    CORS(app, supports_credentials=True)  # Cookie va Session ishlashi uchun supports_credentials shart

    # Blueprintlarni ulash
    app.register_blueprint(auth_bp, url_prefix='/api/auth')
    app.register_blueprint(notes_bp, url_prefix='/api/notes')

    # Global xatoliklar (JSON formatida)
    @app.errorhandler(404)
    def not_found(e):
        return jsonify({'error': 'Resurs topilmadi'}), 404

    @app.errorhandler(500)
    def server_error(e):
        return jsonify({'error': 'Tizim xatoligi yuz berdi'}), 500

    with app.app_context():
        db.create_all()

    return app

from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
