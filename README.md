


from flask import Flask, render_template, request, send_file, redirect, url_for, session
import mysql.connector
import segno
import os
from datetime import datetime, timedelta
import socket
from flask_mail import Mail, Message
from math import radians, cos, sin, asin, sqrt

app = Flask(__name__)
app.secret_key = 'qr_attendance_secret_key_2026'

# ===================== MAIL CONFIG =====================
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'alhassangadafi26@gmail.com'
app.config['MAIL_PASSWORD'] = 'tsjx soxt tyox ivvg'
app.config['MAIL_DEFAULT_SENDER'] = 'alhassangadafi26@gmail.com'

mail = Mail(app)

# ===================== DATABASE =====================
db = mysql.connector.connect(
    host="localhost",
    user="root",
    password="0509050023",
    database="qr_attendance_db",
    buffered=True
)

QR_FOLDER = os.path.join('static', 'qr_codes')
os.makedirs(QR_FOLDER, exist_ok=True)

# ===================== HAVERSINE DISTANCE FUNCTION =====================
def haversine(lat1, lon1, lat2, lon2):
    lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1 
    dlon = lon2 - lon1 
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
    c = 2 * asin(sqrt(a)) 
    r = 6371000  # Radius of Earth in meters
    return c * r

# ===================== HOME =====================
@app.route('/')
def home():
    return render_template('index.html')

# ===================== LECTURER =====================
@app.route('/lecturer/login', methods=['GET', 'POST'])
def lecturer_login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')

        cur = db.cursor(dictionary=True)
        cur.execute("SELECT * FROM lecturers WHERE email=%s AND password=%s", (email, password))
        lecturer = cur.fetchone()
        cur.close()

        if lecturer:
            session['lecturer_logged_in'] = True
            session['lecturer_id'] = lecturer.get('id')
            session['lecturer_name'] = lecturer.get('fullname')
            return redirect('/lecturer/dashboard')
        else:
            error = "Invalid Lecturer Email or Password"
            return render_template('lecturer_login.html', error=error)

    return render_template('lecturer_login.html', error=None)


@app.route('/lecturer/dashboard')
def lecturer_dashboard():
    if 'lecturer_logged_in' not in session:
        return redirect('/lecturer/login')
    return render_template('lecturer_dashboard.html')


@app.route('/lecturer/generate_qr', methods=['GET', 'POST'])
def lecturer_generate_qr():
    if 'lecturer_logged_in' not in session:
        return redirect('/lecturer/login')

    if request.method == 'POST':
        course_code = request.form.get('course_code')
        duration = int(request.form.get('duration', 15))

        if not course_code:
            return "Course Code is required", 400

        expires_at = datetime.now() + timedelta(minutes=duration)
        
        public_url = "  https://dab-had-molecular.ngrok-free.dev"
        
        qr_data = f"{public_url}/mark_attendance?course={course_code}&expires={expires_at.timestamp()}"

        qr = segno.make_qr(qr_data)
        filename = f"session_{course_code}_{datetime.now().strftime('%H%M')}.png"
        filepath = os.path.join(QR_FOLDER, filename)
        qr.save(filepath, scale=8)

        return render_template('generate_qr.html', 
                             qr_image=filename, 
                             course_code=course_code,
                             expires_at=expires_at.strftime("%H:%M"),
                             public_url=public_url)

    return render_template('generate_qr.html')


# ===================== STUDENT =====================
@app.route('/student/login', methods=['GET', 'POST'])
def student_login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')

        cur = db.cursor(dictionary=True)
        cur.execute("SELECT * FROM students WHERE email=%s AND password=%s", (email, password))
        student = cur.fetchone()
        cur.close()

        if student:
            session['student_logged_in'] = True
            session['student_id'] = student['student_id']
            session['student_name'] = student['fullname']
            return redirect('/student/dashboard')
        else:
            error = "Invalid Student Email or Password"
            return render_template('student_login.html', error=error)

    return render_template('student_login.html', error=None)


@app.route('/student/dashboard')
def student_dashboard():
    if 'student_logged_in' not in session:
        return redirect('/student/login')
    
    student_id = session.get('student_id')
    
    cur = db.cursor(dictionary=True)
    cur.execute("SELECT COUNT(*) as attended FROM attendance WHERE student_id = %s", (student_id,))
    result = cur.fetchone()
    attended = result['attended'] if result else 0
    
    total_classes = 20
    percentage = round((attended / total_classes) * 100, 1) if total_classes > 0 else 0
    cur.close()
    
    return render_template('student_dashboard.html', 
                           percentage=percentage,
                           attended=attended,
                           total=total_classes)


# ===================== MARK ATTENDANCE WITH GPS =====================
@app.route('/mark_attendance')
def mark_attendance():
    course_code = request.args.get('course')
    expires_str = request.args.get('expires')
    latitude = request.args.get('lat')
    longitude = request.args.get('lng')

    if not course_code or not expires_str:
        return "<h2>Invalid QR Code</h2>"

    if datetime.now().timestamp() > float(expires_str):
        return "<h2>❌ This QR Code has expired!</h2>"

        if 'student_logged_in' not in session:
            return """
    <h2 style="color:red; text-align:center; margin-top:80px;">
        Please login to your Student Account first
    </h2>
    <p style="text-align:center; font-size:18px;">
        <a href="/student/login">Go to Student Login</a>
    </p>
    """

    student_id = session.get('student_id')

    # GPS Verification (Anti-Proxy)
    if not latitude or not longitude:
        return """
        <h2 style="color:red; text-align:center; margin-top:100px;">
            Location access is required.<br>
            Please enable GPS and scan again.
        </h2>
        """

    # Set your class location (Change these coordinates to your classroom)
    class_lat = 9.4321   # Example latitude
    class_lng = -0.1234  # Example longitude
    allowed_radius = 100  # meters

    distance = haversine(float(latitude), float(longitude), class_lat, class_lng)

    if distance > allowed_radius:
        return f"""
        <h2 style="color:orange; text-align:center; margin-top:100px;">
            ❌ You must be physically present in class.<br>
            Distance: {int(distance)} meters
        </h2>
        """

    # Check duplicate
    cur = db.cursor(dictionary=True)
    cur.execute("""
        SELECT * FROM attendance 
        WHERE student_id=%s AND course_code=%s AND attendance_date=CURDATE()
    """, (student_id, course_code))
    
    if cur.fetchone():
        cur.close()
        return f"""
        <h2 style="color:orange; text-align:center; margin-top:100px;">
            ⚠️ You have already marked attendance for {course_code} today.
        </h2>
        """

    # Mark attendance
    cur.execute("""
        INSERT INTO attendance (student_id, course_code, attendance_date, attendance_time)
        VALUES (%s, %s, CURDATE(), CURTIME())
    """, (student_id, course_code))
    db.commit()

    # Calculate percentage
    cur.execute("SELECT COUNT(*) as total FROM attendance WHERE student_id = %s", (student_id,))
    total_attended = cur.fetchone()['total']
    total_classes = 20
    percentage = round((total_attended / total_classes) * 100, 1) if total_classes > 0 else 0

    # Send Email
    try:
        cur.execute("SELECT fullname, email FROM students WHERE student_id = %s", (student_id,))
        student = cur.fetchone()
        if student and student.get('email'):
            msg = Message("✅ Attendance Marked Successfully", recipients=[student['email']])
            msg.body = f"""
Dear {student['fullname']},

Your attendance has been successfully marked.

Course     : {course_code}
Date       : {datetime.now().strftime('%Y-%m-%d')}
Time       : {datetime.now().strftime('%H:%M:%S')}
Attendance Rate : {percentage}%

Keep up the good work!

QR Attendance System
            """
            mail.send(msg)
    except:
        pass

    cur.close()

    return f"""
    <div style="text-align:center; margin-top:80px; font-family:Arial;">
        <h1 style="color:green;">✅ Attendance Marked Successfully!</h1>
        <h2>Student ID: {student_id}</h2>
        <h3>Course: {course_code}</h3>
        <p>Time: {datetime.now().strftime('%H:%M:%S')}</p>
        <br>
        <h2 style="color:#007bff;">Your Current Attendance Rate: <strong>{percentage}%</strong></h2>
        <p>({total_attended} out of {total_classes} classes)</p>
        <br>
        <a href="/student/dashboard" style="padding:12px 25px; background:#007bff; color:white; text-decoration:none; border-radius:8px;">Go to Dashboard</a>
    </div>
    """


# ===================== OTHER ROUTES =====================
@app.route('/login')
def choose_login():
    return render_template('choose_login.html')


@app.route('/logout')
def logout():
    session.clear()
    return redirect('/')


# ===================== ATTENDANCE RECORDS =====================
@app.route('/attendance')
def attendance():
    if 'lecturer_logged_in' not in session:
        return redirect('/lecturer/login')
    
    cur = db.cursor(dictionary=True)
    cur.execute("SELECT * FROM attendance ORDER BY attendance_date DESC, attendance_time DESC")
    records = cur.fetchall()
    cur.close()
    
    return render_template('attendance.html', records=records)


@app.route('/my_attendance')
def my_attendance():
    if 'student_logged_in' not in session:
        return redirect('/student/login')
    
    student_id = session.get('student_id')
    
    cur = db.cursor(dictionary=True)
    cur.execute("""
        SELECT * FROM attendance 
        WHERE student_id = %s 
        ORDER BY attendance_date DESC, attendance_time DESC
    """, (student_id,))
    records = cur.fetchall()
    
    total_classes = 20
    attended = len(records)
    percentage = round((attended / total_classes) * 100, 1) if total_classes > 0 else 0
    
    cur.close()
    
    return render_template('my_attendance.html', 
                         records=records, 
                         percentage=percentage,
                         attended=attended,
                         total=total_classes)


@app.route('/student/profile')
def student_profile():
    if 'student_logged_in' not in session:
        return redirect('/student/login')
    
    student_id = session.get('student_id')
    
    cur = db.cursor(dictionary=True)
    cur.execute("SELECT * FROM students WHERE student_id = %s", (student_id,))
    student = cur.fetchone()
    cur.close()
    
    return render_template('student_profile.html', student=student)


# ===================== REGISTER =====================
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        student_id = request.form.get('student_id')
        fullname = request.form.get('fullname')
        email = request.form.get('email')
        password = request.form.get('password')

        cur = db.cursor(dictionary=True)
        try:
            cur.execute("""
                INSERT INTO students (student_id, fullname, email, password) 
                VALUES (%s, %s, %s, %s)
            """, (student_id, fullname, email, password))
            db.commit()
            cur.close()
            return "Registration Successful! <a href='/student/login'>Go to Student Login</a>"
        except Exception as e:
            cur.close()
            return f"Error: {str(e)}"

    return render_template('register.html')


# ===================== EXPORT =====================
@app.route('/export_attendance')
def export_attendance():
    if 'lecturer_logged_in' not in session:
        return redirect('/lecturer/login')
    
    cur = db.cursor(dictionary=True)
    cur.execute("""
        SELECT a.attendance_date AS Date, 
               a.attendance_time AS Time, 
               a.student_id AS 'Student ID', 
               s.fullname AS 'Student Name', 
               a.course_code AS 'Course Code'
        FROM attendance a
        LEFT JOIN students s ON a.student_id = s.student_id
        ORDER BY a.attendance_date DESC, a.attendance_time DESC
    """)
    data = cur.fetchall()
    cur.close()

    if not data:
        return "No attendance records found."

    import pandas as pd
    from io import BytesIO

    df = pd.DataFrame(data)
    output = BytesIO()
    df.to_csv(output, index=False, encoding='utf-8')
    output.seek(0)

    return send_file(
        output, 
        as_attachment=True, 
        download_name='attendance_report.csv',
        mimetype='text/csv'
    )


@app.route('/lecturer/courses')
def manage_courses():
    if 'lecturer_logged_in' not in session:
        return redirect('/lecturer/login')
    return render_template('manage_courses.html')


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)

