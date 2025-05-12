from flask import Flask, render_template, request, redirect, url_for, session, flash
import sqlite3
import os
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from io import BytesIO
import base64
from datetime import datetime
import random

app = Flask(__name__)
app.secret_key = os.urandom(24)

def init_db():
    conn = sqlite3.connect('quiz_app.db')
    c = conn.cursor()
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        passkey TEXT NOT NULL,
        role TEXT NOT NULL,
        remarks TEXT
    )
    ''')
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS subjects (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT UNIQUE NOT NULL
    )
    ''')
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS chapters (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        subject_id INTEGER NOT NULL,
        name TEXT NOT NULL,
        FOREIGN KEY (subject_id) REFERENCES subjects (id),
        UNIQUE(subject_id, name)
    )
    ''')
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS questions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        subject_id INTEGER NOT NULL,
        chapter_id INTEGER NOT NULL,
        question_text TEXT NOT NULL,
        option_a TEXT NOT NULL,
        option_b TEXT NOT NULL,
        option_c TEXT NOT NULL,
        option_d TEXT NOT NULL,
        correct_answer TEXT NOT NULL,
        FOREIGN KEY (subject_id) REFERENCES subjects (id),
        FOREIGN KEY (chapter_id) REFERENCES chapters (id)
    )
    ''')
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS quiz_attempts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        subject_id INTEGER NOT NULL,
        total_questions INTEGER NOT NULL,
        correct_answers INTEGER NOT NULL,
        accuracy REAL NOT NULL,
        date_taken TIMESTAMP NOT NULL,
        FOREIGN KEY (user_id) REFERENCES users (id),
        FOREIGN KEY (subject_id) REFERENCES subjects (id)
    )
    ''')
    
    c.execute('''
    CREATE TABLE IF NOT EXISTS quiz_chapters (
        quiz_id INTEGER NOT NULL,
        chapter_id INTEGER NOT NULL,
        FOREIGN KEY (quiz_id) REFERENCES quiz_attempts (id),
        FOREIGN KEY (chapter_id) REFERENCES chapters (id),
        PRIMARY KEY (quiz_id, chapter_id)
    )
    ''')
    
    c.execute("SELECT * FROM users WHERE username = 'admin'")
    if not c.fetchone():
        c.execute("INSERT INTO users (username, passkey, role) VALUES (?, ?, ?)",
                 ('admin', 'admin123', 'admin'))
    
    conn.commit()
    conn.close()

init_db()

def get_db_connection():
    conn = sqlite3.connect('quiz_app.db')
    conn.row_factory = sqlite3.Row
    return conn

@app.route('/')
def home():
    return render_template('home.html')

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    passkey = request.form['passkey']
    
    conn = get_db_connection()
    user = conn.execute('SELECT * FROM users WHERE username = ? AND passkey = ?',
                        (username, passkey)).fetchone()
    conn.close()
    
    if user:
        session['user_id'] = user['id']
        session['username'] = user['username']
        session['role'] = user['role']
        
        if user['role'] == 'admin':
            return redirect(url_for('admin_dashboard'))
        else:
            return redirect(url_for('user_dashboard'))
    else:
        flash('Invalid username or passkey', 'error')
        return redirect(url_for('home'))

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('home'))


@app.route('/admin/dashboard')
def admin_dashboard():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    user_count = conn.execute('SELECT COUNT(*) as count FROM users WHERE role = "user"').fetchone()['count']
    
    subject_count = conn.execute('SELECT COUNT(*) as count FROM subjects').fetchone()['count']
    
    question_count = conn.execute('SELECT COUNT(*) as count FROM questions').fetchone()['count']
    
    quiz_count = conn.execute('SELECT COUNT(*) as count FROM quiz_attempts').fetchone()['count']
    
    recent_quizzes = conn.execute('''
        SELECT qa.id, u.username, s.name as subject, qa.total_questions, qa.accuracy, qa.date_taken
        FROM quiz_attempts qa
        JOIN users u ON qa.user_id = u.id
        JOIN subjects s ON qa.subject_id = s.id
        ORDER BY qa.date_taken DESC LIMIT 5
    ''').fetchall()
    
    conn.close()
    
    return render_template('admin_dashboard.html', 
                          user_count=user_count,
                          subject_count=subject_count,
                          question_count=question_count,
                          quiz_count=quiz_count,
                          recent_quizzes=recent_quizzes)

@app.route('/admin/manage_users')
def manage_users():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    search = request.args.get('search', '')
    
    conn = get_db_connection()
    if search:
        users = conn.execute('SELECT * FROM users WHERE username LIKE ? AND role = "user" ORDER BY username',
                           (f'%{search}%',)).fetchall()
    else:
        users = conn.execute('SELECT * FROM users WHERE role = "user" ORDER BY username').fetchall()
    conn.close()
    
    return render_template('manage_users.html', users=users, search=search)

@app.route('/admin/add_user', methods=['GET', 'POST'])
def add_user():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    if request.method == 'POST':
        username = request.form['username']
        passkey = request.form['passkey']
        remarks = request.form.get('remarks', '')
        
        conn = get_db_connection()
        try:
            conn.execute('INSERT INTO users (username, passkey, role, remarks) VALUES (?, ?, ?, ?)',
                       (username, passkey, 'user', remarks))
            conn.commit()
            flash('User added successfully', 'success')
        except sqlite3.IntegrityError:
            flash('Username already exists', 'error')
        finally:
            conn.close()
        
        return redirect(url_for('manage_users'))
    
    return render_template('add_user.html')

@app.route('/admin/edit_user/<int:user_id>', methods=['GET', 'POST'])
def edit_user(user_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    user = conn.execute('SELECT * FROM users WHERE id = ?', (user_id,)).fetchone()
    
    if not user:
        conn.close()
        flash('User not found', 'error')
        return redirect(url_for('manage_users'))
    
    if request.method == 'POST':
        passkey = request.form['passkey']
        remarks = request.form.get('remarks', '')
        
        conn.execute('UPDATE users SET passkey = ?, remarks = ? WHERE id = ?',
                   (passkey, remarks, user_id))
        conn.commit()
        conn.close()
        
        flash('User updated successfully', 'success')
        return redirect(url_for('manage_users'))
    
    conn.close()
    return render_template('edit_user.html', user=user)

@app.route('/admin/delete_user/<int:user_id>', methods=['POST'])
def delete_user(user_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    conn.execute('DELETE FROM users WHERE id = ? AND role = "user"', (user_id,))
    conn.commit()
    conn.close()
    
    flash('User deleted successfully', 'success')
    return redirect(url_for('manage_users'))

@app.route('/admin/manage_subjects')
def manage_subjects():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    search = request.args.get('search', '')
    
    conn = get_db_connection()
    if search:
        subjects = conn.execute('SELECT * FROM subjects WHERE name LIKE ? ORDER BY name',
                              (f'%{search}%',)).fetchall()
    else:
        subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    conn.close()
    
    return render_template('manage_subjects.html', subjects=subjects, search=search)

@app.route('/admin/add_subject', methods=['GET', 'POST'])
def add_subject():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    if request.method == 'POST':
        name = request.form['name']
        
        conn = get_db_connection()
        try:
            conn.execute('INSERT INTO subjects (name) VALUES (?)', (name,))
            conn.commit()
            flash('Subject added successfully', 'success')
        except sqlite3.IntegrityError:
            flash('Subject already exists', 'error')
        finally:
            conn.close()
        
        return redirect(url_for('manage_subjects'))
    
    return render_template('add_subject.html')

@app.route('/admin/edit_subject/<int:subject_id>', methods=['GET', 'POST'])
def edit_subject(subject_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    subject = conn.execute('SELECT * FROM subjects WHERE id = ?', (subject_id,)).fetchone()
    
    if not subject:
        conn.close()
        flash('Subject not found', 'error')
        return redirect(url_for('manage_subjects'))
    
    if request.method == 'POST':
        name = request.form['name']
        
        try:
            conn.execute('UPDATE subjects SET name = ? WHERE id = ?', (name, subject_id))
            conn.commit()
            flash('Subject updated successfully', 'success')
        except sqlite3.IntegrityError:
            flash('Subject name already exists', 'error')
        finally:
            conn.close()
        
        return redirect(url_for('manage_subjects'))
    
    conn.close()
    return render_template('edit_subject.html', subject=subject)

@app.route('/admin/delete_subject/<int:subject_id>', methods=['POST'])
def delete_subject(subject_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    chapters = conn.execute('SELECT * FROM chapters WHERE subject_id = ?', (subject_id,)).fetchone()
    if chapters:
        conn.close()
        flash('Cannot delete subject with associated chapters', 'error')
        return redirect(url_for('manage_subjects'))
    
    conn.execute('DELETE FROM subjects WHERE id = ?', (subject_id,))
    conn.commit()
    conn.close()
    
    flash('Subject deleted successfully', 'success')
    return redirect(url_for('manage_subjects'))

@app.route('/admin/manage_chapters')
def manage_chapters():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    subject_id = request.args.get('subject_id', type=int)
    search = request.args.get('search', '')
    
    conn = get_db_connection()
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    if subject_id:
        if search:
            chapters = conn.execute('''
                SELECT c.*, s.name as subject_name 
                FROM chapters c
                JOIN subjects s ON c.subject_id = s.id
                WHERE c.subject_id = ? AND c.name LIKE ?
                ORDER BY c.name
            ''', (subject_id, f'%{search}%')).fetchall()
        else:
            chapters = conn.execute('''
                SELECT c.*, s.name as subject_name 
                FROM chapters c
                JOIN subjects s ON c.subject_id = s.id
                WHERE c.subject_id = ?
                ORDER BY c.name
            ''', (subject_id,)).fetchall()
    else:
        if search:
            chapters = conn.execute('''
                SELECT c.*, s.name as subject_name 
                FROM chapters c
                JOIN subjects s ON c.subject_id = s.id
                WHERE c.name LIKE ?
                ORDER BY s.name, c.name
            ''', (f'%{search}%',)).fetchall()
        else:
            chapters = conn.execute('''
                SELECT c.*, s.name as subject_name 
                FROM chapters c
                JOIN subjects s ON c.subject_id = s.id
                ORDER BY s.name, c.name
            ''').fetchall()
    
    conn.close()
    
    return render_template('manage_chapters.html', 
                          chapters=chapters, 
                          subjects=subjects, 
                          current_subject_id=subject_id,
                          search=search)

@app.route('/admin/add_chapter', methods=['GET', 'POST'])
def add_chapter():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    if not subjects:
        conn.close()
        flash('Please add subjects first', 'error')
        return redirect(url_for('manage_subjects'))
    
    if request.method == 'POST':
        subject_id = request.form['subject_id']
        name = request.form['name']
        
        try:
            conn.execute('INSERT INTO chapters (subject_id, name) VALUES (?, ?)', 
                       (subject_id, name))
            conn.commit()
            flash('Chapter added successfully', 'success')
        except sqlite3.IntegrityError:
            flash('Chapter already exists for this subject', 'error')
        finally:
            conn.close()
        
        return redirect(url_for('manage_chapters', subject_id=subject_id))
    
    conn.close()
    return render_template('add_chapter.html', subjects=subjects)

@app.route('/admin/edit_chapter/<int:chapter_id>', methods=['GET', 'POST'])
def edit_chapter(chapter_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    chapter = conn.execute('''
        SELECT c.*, s.name as subject_name 
        FROM chapters c
        JOIN subjects s ON c.subject_id = s.id
        WHERE c.id = ?
    ''', (chapter_id,)).fetchone()
    
    if not chapter:
        conn.close()
        flash('Chapter not found', 'error')
        return redirect(url_for('manage_chapters'))
    
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    if request.method == 'POST':
        subject_id = request.form['subject_id']
        name = request.form['name']
        
        try:
            conn.execute('UPDATE chapters SET subject_id = ?, name = ? WHERE id = ?', 
                       (subject_id, name, chapter_id))
            conn.commit()
            flash('Chapter updated successfully', 'success')
        except sqlite3.IntegrityError:
            flash('Chapter already exists for this subject', 'error')
        finally:
            conn.close()
        
        return redirect(url_for('manage_chapters', subject_id=subject_id))
    
    conn.close()
    return render_template('edit_chapter.html', chapter=chapter, subjects=subjects)

@app.route('/admin/delete_chapter/<int:chapter_id>', methods=['POST'])
def delete_chapter(chapter_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    chapter = conn.execute('SELECT subject_id FROM chapters WHERE id = ?', (chapter_id,)).fetchone()
    if not chapter:
        conn.close()
        flash('Chapter not found', 'error')
        return redirect(url_for('manage_chapters'))
    
    subject_id = chapter['subject_id']
    
    questions = conn.execute('SELECT * FROM questions WHERE chapter_id = ?', (chapter_id,)).fetchone()
    if questions:
        conn.close()
        flash('Cannot delete chapter with associated questions', 'error')
        return redirect(url_for('manage_chapters', subject_id=subject_id))
    
    conn.execute('DELETE FROM chapters WHERE id = ?', (chapter_id,))
    conn.commit()
    conn.close()
    
    flash('Chapter deleted successfully', 'success')
    return redirect(url_for('manage_chapters', subject_id=subject_id))

@app.route('/admin/manage_questions')
def manage_questions():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    subject_id = request.args.get('subject_id', type=int)
    chapter_id = request.args.get('chapter_id', type=int)
    search = request.args.get('search', '')
    
    conn = get_db_connection()
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    chapters = []
    if subject_id:
        chapters = conn.execute('''
            SELECT * FROM chapters 
            WHERE subject_id = ? 
            ORDER BY name
        ''', (subject_id,)).fetchall()
    
    if subject_id and chapter_id:
        if search:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                WHERE q.subject_id = ? AND q.chapter_id = ? AND q.question_text LIKE ?
                ORDER BY q.id
            ''', (subject_id, chapter_id, f'%{search}%')).fetchall()
        else:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                WHERE q.subject_id = ? AND q.chapter_id = ?
                ORDER BY q.id
            ''', (subject_id, chapter_id)).fetchall()
    elif subject_id:
        if search:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                WHERE q.subject_id = ? AND q.question_text LIKE ?
                ORDER BY c.name, q.id
            ''', (subject_id, f'%{search}%')).fetchall()
        else:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                WHERE q.subject_id = ?
                ORDER BY c.name, q.id
            ''', (subject_id,)).fetchall()
    else:
        if search:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                WHERE q.question_text LIKE ?
                ORDER BY s.name, c.name, q.id
            ''', (f'%{search}%',)).fetchall()
        else:
            questions = conn.execute('''
                SELECT q.*, s.name as subject_name, c.name as chapter_name
                FROM questions q
                JOIN subjects s ON q.subject_id = s.id
                JOIN chapters c ON q.chapter_id = c.id
                ORDER BY s.name, c.name, q.id
            ''').fetchall()
    
    conn.close()
    
    return render_template('manage_questions.html', 
                          questions=questions, 
                          subjects=subjects,
                          chapters=chapters,
                          current_subject_id=subject_id,
                          current_chapter_id=chapter_id,
                          search=search)

@app.route('/admin/add_question', methods=['GET', 'POST'])
def add_question():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    if not subjects:
        conn.close()
        flash('Please add subjects first', 'error')
        return redirect(url_for('manage_subjects'))
    
    chapters = []
    subject_id = request.args.get('subject_id', type=int)
    if subject_id:
        chapters = conn.execute('''
            SELECT * FROM chapters 
            WHERE subject_id = ? 
            ORDER BY name
        ''', (subject_id,)).fetchall()
    
    if request.method == 'POST':
        subject_id = request.form['subject_id']
        chapter_id = request.form['chapter_id']
        question_text = request.form['question_text']
        option_a = request.form['option_a']
        option_b = request.form['option_b']
        option_c = request.form['option_c']
        option_d = request.form['option_d']
        correct_answer = request.form['correct_answer']
        
        conn.execute('''
            INSERT INTO questions 
            (subject_id, chapter_id, question_text, option_a, option_b, option_c, option_d, correct_answer) 
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', (subject_id, chapter_id, question_text, option_a, option_b, option_c, option_d, correct_answer))
        conn.commit()
        conn.close()
        
        flash('Question added successfully', 'success')
        return redirect(url_for('manage_questions', subject_id=subject_id, chapter_id=chapter_id))
    
    conn.close()
    return render_template('add_question.html', 
                          subjects=subjects, 
                          chapters=chapters, 
                          current_subject_id=subject_id)

@app.route('/get_chapters/<int:subject_id>')
def get_chapters(subject_id):
    conn = get_db_connection()
    chapters = conn.execute('''
        SELECT * FROM chapters 
        WHERE subject_id = ? 
        ORDER BY name
    ''', (subject_id,)).fetchall()
    conn.close()
    
    chapters_list = [{'id': chapter['id'], 'name': chapter['name']} for chapter in chapters]
    return {'chapters': chapters_list}

@app.route('/admin/edit_question/<int:question_id>', methods=['GET', 'POST'])
def edit_question(question_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    question = conn.execute('''
        SELECT q.*, s.name as subject_name, c.name as chapter_name
        FROM questions q
        JOIN subjects s ON q.subject_id = s.id
        JOIN chapters c ON q.chapter_id = c.id
        WHERE q.id = ?
    ''', (question_id,)).fetchone()
    
    if not question:
        conn.close()
        flash('Question not found', 'error')
        return redirect(url_for('manage_questions'))
    
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    chapters = conn.execute('''
        SELECT * FROM chapters 
        WHERE subject_id = ? 
        ORDER BY name
    ''', (question['subject_id'],)).fetchall()
    
    if request.method == 'POST':
        subject_id = request.form['subject_id']
        chapter_id = request.form['chapter_id']
        question_text = request.form['question_text']
        option_a = request.form['option_a']
        option_b = request.form['option_b']
        option_c = request.form['option_c']
        option_d = request.form['option_d']
        correct_answer = request.form['correct_answer']
        
        conn.execute('''
            UPDATE questions 
            SET subject_id = ?, chapter_id = ?, question_text = ?, 
                option_a = ?, option_b = ?, option_c = ?, option_d = ?, correct_answer = ?
            WHERE id = ?
        ''', (subject_id, chapter_id, question_text, option_a, option_b, option_c, option_d, correct_answer, question_id))
        conn.commit()
        conn.close()
        
        flash('Question updated successfully', 'success')
        return redirect(url_for('manage_questions', subject_id=subject_id, chapter_id=chapter_id))
    
    conn.close()
    return render_template('edit_question.html', 
                          question=question, 
                          subjects=subjects, 
                          chapters=chapters)

@app.route('/admin/delete_question/<int:question_id>', methods=['POST'])
def delete_question(question_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    question = conn.execute('''
        SELECT subject_id, chapter_id 
        FROM questions 
        WHERE id = ?
    ''', (question_id,)).fetchone()
    
    if not question:
        conn.close()
        flash('Question not found', 'error')
        return redirect(url_for('manage_questions'))
    
    subject_id = question['subject_id']
    chapter_id = question['chapter_id']
    
    conn.execute('DELETE FROM questions WHERE id = ?', (question_id,))
    conn.commit()
    conn.close()
    
    flash('Question deleted successfully', 'success')
    return redirect(url_for('manage_questions', subject_id=subject_id, chapter_id=chapter_id))

@app.route('/admin/view_user_progress')
def view_user_progress():
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    users = conn.execute('''
        SELECT u.id, u.username, 
               COUNT(qa.id) as total_quizzes,
               IFNULL(AVG(qa.accuracy), 0) as avg_accuracy
        FROM users u
        LEFT JOIN quiz_attempts qa ON u.id = qa.user_id
        WHERE u.role = 'user'
        GROUP BY u.id
        ORDER BY u.username
    ''').fetchall()
    
    user_names = []
    accuracies = []
    
    for user in users:
        if user['total_quizzes'] > 0:  
            user_names.append(user['username'])
            accuracies.append(user['avg_accuracy'])
    
    # Create chart
    chart_img = None
    if user_names:
        plt.figure(figsize=(10, 6))
        bars = plt.bar(user_names, accuracies, color='skyblue')
        plt.xlabel('Users')
        plt.ylabel('Average Accuracy (%)')
        plt.title('User Performance Summary')
        plt.ylim(0, 100)
        plt.xticks(rotation=45, ha='right')
        plt.tight_layout()
        
        for bar in bars:
            height = bar.get_height()
            plt.text(bar.get_x() + bar.get_width()/2., height + 1,
                    f'{height:.1f}%', ha='center', va='bottom')
        
        img = BytesIO()
        plt.savefig(img, format='png')
        img.seek(0)
        chart_img = base64.b64encode(img.getvalue()).decode('utf-8')
        plt.close()
    
    conn.close()
    
    return render_template('view_user_progress.html', 
                          users=users, 
                          chart_img=chart_img)


@app.route('/admin/user_details/<int:user_id>')
def user_details(user_id):
    if not session.get('user_id') or session.get('role') != 'admin':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    user = conn.execute('SELECT * FROM users WHERE id = ?', (user_id,)).fetchone()
    
    if not user:
        conn.close()
        flash('User not found', 'error')
        return redirect(url_for('view_user_progress'))
    
    quizzes = conn.execute('''
        SELECT qa.*, s.name as subject_name
        FROM quiz_attempts qa
        JOIN subjects s ON qa.subject_id = s.id
        WHERE qa.user_id = ?
        ORDER BY qa.date_taken DESC
    ''', (user_id,)).fetchall()
    
    subject_data = conn.execute('''
        SELECT s.name as subject_name, 
               COUNT(qa.id) as attempt_count,
               AVG(qa.accuracy) as avg_accuracy
        FROM quiz_attempts qa
        JOIN subjects s ON qa.subject_id = s.id
        WHERE qa.user_id = ?
        GROUP BY qa.subject_id
        ORDER BY avg_accuracy DESC
    ''', (user_id,)).fetchall()
    
    chart_img = None
    if subject_data:
        subjects = []
        accuracies = []
        
        for data in subject_data:
            subjects.append(data['subject_name'])
            accuracies.append(data['avg_accuracy'])
        
        plt.figure(figsize=(10, 6))
        bars = plt.bar(subjects, accuracies, color='lightgreen')
        plt.xlabel('Subjects')
        plt.ylabel('Average Accuracy (%)')
        plt.title(f'Subject Performance for {user["username"]}')
        plt.ylim(0, 100)
        plt.xticks(rotation=45, ha='right')
        plt.tight_layout()
        
        for bar in bars:
            height = bar.get_height()
            plt.text(bar.get_x() + bar.get_width()/2., height + 1,
                    f'{height:.1f}%', ha='center', va='bottom')
        
        img = BytesIO()
        plt.savefig(img, format='png')
        img.seek(0)
        chart_img = base64.b64encode(img.getvalue()).decode('utf-8')
        plt.close()
    
    conn.close()
    
    return render_template('user_details.html', 
                          user=user, 
                          quizzes=quizzes, 
                          subject_data=subject_data,
                          chart_img=chart_img)

@app.route('/user/dashboard')
def user_dashboard():
    if not session.get('user_id') or session.get('role') != 'user':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    quiz_count = conn.execute('''
        SELECT COUNT(*) as count 
        FROM quiz_attempts 
        WHERE user_id = ?
    ''', (session['user_id'],)).fetchone()['count']
    
    avg_accuracy = conn.execute('''
        SELECT IFNULL(AVG(accuracy), 0) as avg_accuracy 
        FROM quiz_attempts 
        WHERE user_id = ?
    ''', (session['user_id'],)).fetchone()['avg_accuracy']
    
    recent_quizzes = conn.execute('''
        SELECT qa.*, s.name as subject_name
        FROM quiz_attempts qa
        JOIN subjects s ON qa.subject_id = s.id
        WHERE qa.user_id = ?
        ORDER BY qa.date_taken DESC LIMIT 5
    ''', (session['user_id'],)).fetchall()
    
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    conn.close()
    
    return render_template('user_dashboard.html', 
                          quiz_count=quiz_count,
                          avg_accuracy=avg_accuracy,
                          recent_quizzes=recent_quizzes,
                          subjects=subjects)

@app.route('/user/list_chapters', methods=['GET', 'POST'])
def list_chapters():
	if not session.get('user_id') or session.get('role') != 'user':
		return redirect(url_for('home'))
	conn = get_db_connection()
if request.method == 'POST':
	subject_id = request.form['subject_id']
chapters_to_be_shown = conn.execute ( '''
	select * 
	from subjects join chapters
	on subjects.id=chapters_subjects.id
	order by name aesc 
	''').fetchall()
	conn.close()
	return render_template('see_chapters.html',
			subject_id=subject_id)




@app.route('/user/start_quiz', methods=['GET', 'POST'])
def start_quiz():
    if not session.get('user_id') or session.get('role') != 'user':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    if request.method == 'POST':
        subject_id = request.form['subject_id']
        chapter_ids = request.form.getlist('chapter_ids')
        num_questions = int(request.form['num_questions'])
        
        if not chapter_ids:
            conn.close()
            flash('Please select at least one chapter', 'error')
            return redirect(url_for('start_quiz'))
        
        questions = []
        for chapter_id in chapter_ids:
            chapter_questions = conn.execute('''
                SELECT * FROM questions 
                WHERE subject_id = ? AND chapter_id = ?
            ''', (subject_id, chapter_id)).fetchall()
            questions.extend(chapter_questions)
        
        if not questions:
            conn.close()
            flash('No questions available for selected chapters', 'error')
            return redirect(url_for('start_quiz'))
        
        if len(questions) > num_questions:
            questions = random.sample(questions, num_questions)
        
        session['quiz_questions'] = [dict(q) for q in questions]
        session['quiz_subject_id'] = subject_id
        session['quiz_chapter_ids'] = chapter_ids
        session['quiz_current_question'] = 0
        session['quiz_correct_answers'] = 0
        session['quiz_start_time'] = datetime.now().timestamp()
        
        conn.close()
        return redirect(url_for('take_quiz'))
    
    subjects = conn.execute('SELECT * FROM subjects ORDER BY name').fetchall()
    
    chapters = []
    if subjects:
        first_subject_id = subjects[0]['id']
        chapters = conn.execute('''
            SELECT * FROM chapters 
            WHERE subject_id = ? 
            ORDER BY name
        ''', (first_subject_id,)).fetchall()
    
    conn.close()
    
    return render_template('start_quiz.html', 
                          subjects=subjects,
                          chapters=chapters)

@app.route('/user/take_quiz', methods=['GET', 'POST'])
def take_quiz():
    if not session.get('user_id') or session.get('role') != 'user' or 'quiz_questions' not in session:
        return redirect(url_for('user_dashboard'))
    
    questions = session['quiz_questions']
    current_index = session['quiz_current_question']
    
    if current_index >= len(questions):
        return redirect(url_for('quiz_results'))
    
    current_question = questions[current_index]
    
    if request.method == 'POST':
        user_answer = request.form.get('answer')
        correct_answer = current_question['correct_answer']
        
        if user_answer == correct_answer:
            session['quiz_correct_answers'] += 1
        
        session['quiz_current_question'] += 1
        return redirect(url_for('take_quiz'))

    question_time_limit = 120  
    start_time = session['quiz_start_time']
    elapsed_time = datetime.now().timestamp() - start_time
    time_per_question = elapsed_time / (current_index + 1) if current_index > 0 else 0
    time_remaining = max(0, question_time_limit - time_per_question)
    
    return render_template('take_quiz.html', 
                          question=current_question,
                          question_number=current_index + 1,
                          total_questions=len(questions),
                          time_remaining=int(time_remaining))

@app.route('/user/quiz_results')
def quiz_results():
    if not session.get('user_id') or session.get('role') != 'user' or 'quiz_questions' not in session:
        return redirect(url_for('user_dashboard'))
    
    total_questions = len(session['quiz_questions'])
    correct_answers = session['quiz_correct_answers']
    accuracy = (correct_answers / total_questions) * 100 if total_questions > 0 else 0
    
    conn = get_db_connection()
    
    cursor = conn.execute('''
        INSERT INTO quiz_attempts 
        (user_id, subject_id, total_questions, correct_answers, accuracy, date_taken)
        VALUES (?, ?, ?, ?, ?, ?)
    ''', (session['user_id'], session['quiz_subject_id'], total_questions, 
          correct_answers, accuracy, datetime.now()))
    
    quiz_id = cursor.lastrowid
    
    for chapter_id in session['quiz_chapter_ids']:
        conn.execute('''
            INSERT INTO quiz_chapters (quiz_id, chapter_id)
            VALUES (?, ?)
        ''', (quiz_id, chapter_id))
    
    conn.commit()
    
    subject = conn.execute('SELECT name FROM subjects WHERE id = ?', 
                         (session['quiz_subject_id'],)).fetchone()
    
    chapters = conn.execute('''
        SELECT name FROM chapters 
        WHERE id IN ({})
    '''.format(','.join(['?'] * len(session['quiz_chapter_ids']))), 
                          session['quiz_chapter_ids']).fetchall()
    
    conn.close()
    
    plt.figure(figsize=(8, 6))
    plt.pie([correct_answers, total_questions - correct_answers], 
           labels=['Correct', 'Incorrect'],
           colors=['#4CAF50', '#F44336'],
           autopct='%1.1f%%',
           startangle=90)
    plt.axis('equal')
    plt.title('Quiz Results')
    
    img = BytesIO()
    plt.savefig(img, format='png')
    img.seek(0)
    chart_img = base64.b64encode(img.getvalue()).decode('utf-8')
    plt.close()
    
    session.pop('quiz_questions', None)
    session.pop('quiz_subject_id', None)
    session.pop('quiz_chapter_ids', None)
    session.pop('quiz_current_question', None)
    session.pop('quiz_correct_answers', None)
    session.pop('quiz_start_time', None)
    
    return render_template('quiz_results.html',
                          total_questions=total_questions,
                          correct_answers=correct_answers,
                          accuracy=accuracy,
                          subject_name=subject['name'],
                          chapter_names=[chapter['name'] for chapter in chapters],
                          chart_img=chart_img)

@app.route('/user/view_previous_attempts')
def view_previous_attempts():
    if not session.get('user_id') or session.get('role') != 'user':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    quizzes = conn.execute('''
        SELECT qa.*, s.name as subject_name
        FROM quiz_attempts qa
        JOIN subjects s ON qa.subject_id = s.id
        WHERE qa.user_id = ?
        ORDER BY qa.date_taken DESC
    ''', (session['user_id'],)).fetchall()
    
    chart_img = None
    if quizzes:
        recent_quizzes = quizzes[:10]
        recent_quizzes.reverse()
        
        dates = [datetime.strptime(q['date_taken'], '%Y-%m-%d %H:%M:%S.%f').strftime('%m/%d %H:%M') 
                for q in recent_quizzes]
        accuracies = [q['accuracy'] for q in recent_quizzes]

        plt.figure(figsize=(10, 6))
        plt.plot(dates, accuracies, marker='o', linestyle='-', color='blue')
        plt.xlabel('Quiz Date')
        plt.ylabel('Accuracy (%)')
        plt.title('Quiz Performance Over Time')
        plt.ylim(0, 100)
        plt.grid(True, linestyle='--', alpha=0.7)
        plt.xticks(rotation=45)
        plt.tight_layout()
        
        img = BytesIO()
        plt.savefig(img, format='png')
        img.seek(0)
        chart_img = base64.b64encode(img.getvalue()).decode('utf-8')
        plt.close()
    
    conn.close()
    
    return render_template('view_previous_attempts.html', 
                          quizzes=quizzes,
                          chart_img=chart_img)

@app.route('/user/quiz_details/<int:quiz_id>')
def quiz_details(quiz_id):
    if not session.get('user_id') or session.get('role') != 'user':
        return redirect(url_for('home'))
    
    conn = get_db_connection()
    
    quiz = conn.execute('''
        SELECT qa.*, s.name as subject_name
        FROM quiz_attempts qa
        JOIN subjects s ON qa.subject_id = s.id
        WHERE qa.id = ? AND qa.user_id = ?
    ''', (quiz_id, session['user_id'])).fetchone()
    
    if not quiz:
        conn.close()
        flash('Quiz not found', 'error')
        return redirect(url_for('view_previous_attempts'))
    
    chapters = conn.execute('''
        SELECT c.name
        FROM quiz_chapters qc
        JOIN chapters c ON qc.chapter_id = c.id
        WHERE qc.quiz_id = ?
    ''', (quiz_id,)).fetchall()
    
    conn.close()
    
    plt.figure(figsize=(8, 6))
    plt.pie([quiz['correct_answers'], quiz['total_questions'] - quiz['correct_answers']], 
           labels=['Correct', 'Incorrect'],
           colors=['#4CAF50', '#F44336'],
           autopct='%1.1f%%',
           startangle=90)
    plt.axis('equal')
    plt.title('Quiz Results')
    
    img = BytesIO()
    plt.savefig(img, format='png')
    img.seek(0)
    chart_img = base64.b64encode(img.getvalue()).decode('utf-8')
    plt.close()
    
    return render_template('quiz_details.html',
                          quiz=quiz,
                          chapter_names=[chapter['name'] for chapter in chapters],
                          chart_img=chart_img)

if __name__ == '__main__':
    app.run(debug=True)

