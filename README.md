import cv2
import os
from flask import Flask, request, render_template
from datetime import date, datetime
import numpy as np
from sklearn.neighbors import KNeighborsClassifier
import pandas as pd
import joblib

app = Flask(__name__,template_folder='templates')


nimgs = 10


imgBackground = cv2.imread(r"E:\test\background.png\WIN_20250409_21_04_25_Pro.png")


if imgBackground is None:
    raise FileNotFoundError("background.png not found in current directory.")

datetoday = date.today().strftime("%m_%d_%y")
datetoday2 = date.today().strftime("%d-%B-%Y")


face_detector = cv2.CascadeClassifier(r'E:\test\haarcascade_frontalface_default.xml\haarcascade_frontalface_default.xml')
if face_detector.empty():
    raise FileNotFoundError("Haarcascade XML not found or corrupt.")


os.makedirs('Attendance', exist_ok=True)
os.makedirs('static', exist_ok=True)
os.makedirs('static/faces', exist_ok=True)


csv_path = f'Attendance/Attendance-{datetoday}.csv'
if not os.path.exists(csv_path):
    with open(csv_path, 'w') as f:
        f.write('Name,Roll,Time')

def totalreg():
    return len(os.listdir('static/faces'))

def extract_faces(img):
    try:
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        face_points = face_detector.detectMultiScale(gray, 1.2, 5, minSize=(20, 20))
        return face_points
    except:
        return []

def identify_face(facearray):
    model = joblib.load('static/face_recognition_model.pkl')
    return model.predict(facearray)

def train_model():
    faces = []
    labels = []
    for user in os.listdir('static/faces'):
        for imgname in os.listdir(f'static/faces/{user}'):
            img = cv2.imread(f'static/faces/{user}/{imgname}')
            if img is None:
                continue
            resized_face = cv2.resize(img, (50, 50))
            faces.append(resized_face.ravel())
            labels.append(user)
    faces = np.array(faces)
    knn = KNeighborsClassifier(n_neighbors=5)
    knn.fit(faces, labels)
    joblib.dump(knn, 'static/face_recognition_model.pkl')

def extract_attendance():
    df = pd.read_csv(csv_path)
    return df['Name'], df['Roll'], df['Time'], len(df)

def add_attendance(name):
    username, userid = name.split('_')
    current_time = datetime.now().strftime("%H:%M:%S")
    df = pd.read_csv(csv_path)
    if int(userid) not in df['Roll'].astype(int).values:
        with open(csv_path, 'a') as f:
            f.write(f'\n{username},{userid},{current_time}')

@app.route('/')
def home():
    names, rolls, times, l = extract_attendance()
    return render_template('home.html', names=names, rolls=rolls, times=times, l=l, totalreg=totalreg(), datetoday2=datetoday2)

@app.route('/start', methods=['GET'])
def start():
    names, rolls, times, l = extract_attendance()

    if 'face_recognition_model.pkl' not in os.listdir('static'):
        return render_template('home.html', names=names, rolls=rolls, times=times, l=l, totalreg=totalreg(), datetoday2=datetoday2, mess='Train the model by adding a face first.')

    cap = cv2.VideoCapture(0)
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        faces = extract_faces(frame)
        if len(faces) > 0:
            (x, y, w, h) = faces[0]
            cv2.rectangle(frame, (x, y), (x + w, y + h), (86, 32, 251), 1)
            face = cv2.resize(frame[y:y + h, x:x + w], (50, 50))
            identified_person = identify_face(face.reshape(1, -1))[0]
            add_attendance(identified_person)
            cv2.putText(frame, f'{identified_person}', (x, y - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, (255,255,255), 2)
        imgBackground[162:642, 55:695] = frame
        cv2.imshow('Attendance', imgBackground)
        if cv2.waitKey(1) == 27:  # ESC key to exit
            break
    cap.release()
    cv2.destroyAllWindows()
    names, rolls, times, l = extract_attendance()
    return render_template('home.html', names=names, rolls=rolls, times=times, l=l, totalreg=totalreg(), datetoday2=datetoday2)

@app.route('/add', methods=['GET', 'POST'])
def add():
    if request.method == 'POST':
        newusername = request.form['newusername']
        newuserid = request.form['newuserid']
        folder = f'static/faces/{newusername}_{newuserid}'
        os.makedirs(folder, exist_ok=True)
        cap = cv2.VideoCapture(0)
        i, j = 0, 0
        while True:
            ret, frame = cap.read()
            if not ret:
                break
            faces = extract_faces(frame)
            for (x, y, w, h) in faces:
                cv2.rectangle(frame, (x, y), (x + w, y + h), (255, 0, 20), 2)
                if j % 5 == 0 and i < nimgs:
                    img_name = f"{newusername}_{i}.jpg"
                    cv2.imwrite(f"{folder}/{img_name}", frame[y:y + h, x:x + w])
                    i += 1
                j += 1
            cv2.putText(frame, f'Images Captured: {i}/{nimgs}', (30, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 0, 20), 2)
            cv2.imshow("Adding New User", frame)
            if cv2.waitKey(1) == 27 or i >= nimgs:
                break
        cap.release()
        cv2.destroyAllWindows()
        train_model()
    names, rolls, times, l = extract_attendance()
    return render_template('home.html', names=names, rolls=rolls, times=times, l=l, totalreg=totalreg(), datetoday2=datetoday2)

if __name__ == '__main__':
    app.run(debug=True, use_reloader=True)
