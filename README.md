# Car-Rental-App
from flask import Flask, render_template, request, redirect, url_for, session,flash
from datetime import datetime
import pymysql

app = Flask(__name__)
app.secret_key = 'your_secret_key'  # Add a secret key for session management

# Global DB config dictionary for reuse
db_config = {
    'host': 'localhost',
    'user': 'root',
    'password': 'root',
    'database': 'car-rental',
    'cursorclass': pymysql.cursors.DictCursor
}

# Function to validate user credentials
def validate_user(username, password):
    connection = pymysql.connect(**db_config)
    cursor = connection.cursor()
    query = "SELECT * FROM users WHERE user_name = %s AND user_pwd = %s"
    cursor.execute(query, (username, password))
    result = cursor.fetchone()
    cursor.close()
    connection.close()

    if result:
        session['user_id'] = result['user_id']  # Save user ID to session (ensure column name matches)
        session['username'] = result['user_name']
        print(f"User logged in: {session['user_id']}")  # Debug log
        return True
    return False

@app.route('/')
def home():
    return render_template('home.html')  # Render the home page

@app.route('/book-now')
def book_now():
    if 'user_id' not in session:
        return redirect(url_for('login'))  # Redirect to login if not logged in
    return redirect(url_for('mainpage'))  # Redirect to main page if logged in

@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']

        if validate_user(username, password):
            return redirect(url_for('mainpage'))  # Redirect to main page on successful login
        else:
            error = "Invalid username or password. Please try again."

    return render_template('login.html', error=error)  # Render the login page with error message

@app.route('/register', methods=['GET', 'POST'])
def register():
    message = None
    if request.method == 'POST':
        # Retrieve form data
        username = request.form['username']
        email = request.form['email']
        password = request.form['password']

        try:
            # Connect to the database
            connection = pymysql.connect(**db_config)
            with connection.cursor() as cursor:
                # Insert user data into the database
                sql = """
                INSERT INTO users (user_name, email, user_pwd)
                VALUES (%s, %s, %s)
                """
                cursor.execute(sql, (username, email, password))
                connection.commit()
                message = 'Registration successful. Please log in.'
                return redirect(url_for('login'))  # Redirect to login page after successful registration
        except pymysql.MySQLError as e:
            print("Error:", e)  # Log the error for debugging
            message = 'Registration failed. Please try again.'
        finally:
            connection.close()

    return render_template('register.html', message=message)  # Render the register page with message

@app.route('/mainpage')
def mainpage():
    if 'user_id' in session:  # Check if the user is logged in
        try:
            connection = pymysql.connect(**db_config)
            with connection.cursor() as cursor:
                # Fetch all car data including image URLs
                sql = "SELECT * FROM cars WHERE status = 'available'"
                cursor.execute(sql)
                cars = cursor.fetchall()  # Fetch all rows as a list of dictionaries
        finally:
            connection.close()

        # Pass the car data to the template
        return render_template('mainpage.html', cars=cars)
    else:
        return redirect(url_for('login'))  # Redirect to login if not logged in


@app.route('/booking/<int:car_id>', methods=['GET', 'POST'])
def booking(car_id):
    try:
        connection = pymysql.connect(**db_config)
        with connection.cursor() as cursor:
            # Fetch car details
            sql = "SELECT * FROM cars WHERE id = %s"
            cursor.execute(sql, (car_id,))
            car = cursor.fetchone()

            if not car:
                flash("Car not found.", "error")
                return redirect(url_for('mainpage'))

        if request.method == 'POST':
            # Get user_id from the session
            user_id = session.get('user_id')
            if not user_id:
                flash("You must be logged in to book a car.", "error")
                return redirect(url_for('login'))

            # Get form data
            pickup_location = request.form['pickup_location']
            dropoff_location = request.form['dropoff_location']
            pickup_date = request.form['pickup_date']
            dropoff_date = request.form['dropoff_date']
            booking_status = request.form['booking_status']

            # Calculate the number of days
            try:
                pickup_date_obj = datetime.strptime(pickup_date, '%Y-%m-%d')
                dropoff_date_obj = datetime.strptime(dropoff_date, '%Y-%m-%d')
                days = (dropoff_date_obj - pickup_date_obj).days

                if days <= 0:
                    flash("Dropoff date must be after the pickup date.", "error")
                    return redirect(url_for('booking', car_id=car_id))

            except ValueError:
                flash("Invalid date format. Please use YYYY-MM-DD.", "error")
                return redirect(url_for('booking', car_id=car_id))

            # Calculate total amount
            price_per_day = car['price_per_day']
            total_amount = days * price_per_day

            # Insert booking details into the database
            try:
                with connection.cursor() as cursor:
                    sql = """
                        INSERT INTO bookings (user_id, car_id, pickup_location, dropoff_location, pickup_date, dropoff_date, total_amount, booking_status, created_at)
                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, NOW())
                    """
                    cursor.execute(sql, (user_id, car_id, pickup_location, dropoff_location, pickup_date, dropoff_date, total_amount, booking_status))
                    connection.commit()

                    # Redirect to completed bookings page
                    return redirect(url_for('completed_bookings'))

            except pymysql.MySQLError as e:
                flash(f"Database error: {e}", "error")
                return redirect(url_for('booking', car_id=car_id))

    except pymysql.MySQLError as e:
        flash(f"Database connection error: {e}", "error")
        return redirect(url_for('mainpage'))

    finally:
        connection.close()

    # Render the booking form with car details
    return render_template('booking.html', car=car)

@app.route('/completed_bookings')
def completed_bookings():
    if 'user_id' not in session:
        flash("You must be logged in to view completed bookings.", "error")
        return redirect(url_for('login'))

    try:
        connection = pymysql.connect(**db_config)
        with connection.cursor(pymysql.cursors.DictCursor) as cursor:
            # Fetch completed bookings from the database
            sql = """
                SELECT users.user_name,bookings.user_id, bookings.car_id, bookings.pickup_location, 
                       bookings.dropoff_location, bookings.pickup_date, bookings.dropoff_date, 
                       bookings.total_amount
                FROM bookings
                JOIN users ON bookings.user_id = users.user_id
                WHERE bookings.booking_status = 'Completed'
            """
            cursor.execute(sql)
            bookings = cursor.fetchall()
    except pymysql.MySQLError as e:
        flash(f"Database error: {e}", "error")
        bookings = []
    finally:
        connection.close()

    # Render the completed bookings page
    return render_template('completed_bookings.html', bookings=bookings)

@app.route('/logout')
def logout():
    session.clear()  # Clear the session
    return redirect(url_for('login'))  # Redirect to login page

if __name__ == '__main__':
    app.run(debug=True)
