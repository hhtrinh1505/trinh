import cv2
import time
import numpy as np
import mediapipe as mp
import pyautogui
import screen_brightness_control as sbc
from ctypes import cast, POINTER
from comtypes import CLSCTX_ALL
from pycaw.pycaw import AudioUtilities, IAudioEndpointVolume
import math
import os
from datetime import datetime

print("--- DANG KHOI DONG HE THONG (FULL SCREEN CONTROL) ---")

if not os.path.exists("Screenshots"):
    os.makedirs("Screenshots")

# --- 1. CẤU HÌNH CAMERA HD ---
wCam, hCam = 1280, 720  
cap = cv2.VideoCapture(0)
cap.set(3, wCam)
cap.set(4, hCam)

if not cap.isOpened():
    cap = cv2.VideoCapture(0)

wScr, hScr = pyautogui.size()

# --- CHỈNH SỬA: Đặt khung giới hạn về 0 (Dùng toàn màn hình) ---
frameR = 0  

# --- 2. KHOI TAO MEDIAPIPE ---
mpHands = mp.solutions.hands
hands = mpHands.Hands(static_image_mode=False,
                      max_num_hands=1,
                      min_detection_confidence=0.7,
                      min_tracking_confidence=0.5)
mpDraw = mp.solutions.drawing_utils

# --- 3. KHOI TAO AM THANH ---
try:
    devices = AudioUtilities.GetSpeakers()
    interface = devices.Activate(IAudioEndpointVolume._iid_, CLSCTX_ALL, None)
    volume = cast(interface, POINTER(IAudioEndpointVolume))
    volRange = volume.GetVolumeRange()
    minVol = volRange[0]
    maxVol = volRange[1]
except:
    minVol, maxVol = -65, 0

# --- BIEN TRANG THAI ---
mode = 0 
modes_name = ["MOUSE", "VOLUME", "BRIGHTNESS"] 
smoothening = 5 # Giam do muot xuong mot chut de chuot nhay hon khi di chuyen xa
plocX, plocY = 0, 0
clocX, clocY = 0, 0
last_mode_switch_time = 0
last_screenshot_time = 0
screenshot_notification_start = 0

def get_fingers_status(lmList, tipIds):
    fingers = []
    # Ngón cái
    if lmList[tipIds[0]][1] > lmList[tipIds[0] - 1][1]: 
        fingers.append(1)
    else:
        fingers.append(0)
    # 4 ngón còn lại
    for id in range(1, 5):
        if lmList[tipIds[id]][2] < lmList[tipIds[id] - 2][2]:
            fingers.append(1)
        else:
            fingers.append(0)
    return fingers

cv2.namedWindow("He thong dieu khien bang cu chi", cv2.WINDOW_NORMAL)
cv2.resizeWindow("He thong dieu khien bang cu chi", 1280, 720) 

print("--- HE THONG SAN SANG! ---")

while True:
    success, img = cap.read()
    if not success: break
    
    img = cv2.flip(img, 1)
    
    # (Da xoa lenh ve khung mau tim o day)
    
    imgRGB = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    results = hands.process(imgRGB)
    
    lmList = []
    if results.multi_hand_landmarks:
        for handLms in results.multi_hand_landmarks:
            mpDraw.draw_landmarks(img, handLms, mpHands.HAND_CONNECTIONS)
            for id, lm in enumerate(handLms.landmark):
                h, w, c = img.shape
                cx, cy = int(lm.x * w), int(lm.y * h)
                lmList.append([id, cx, cy])

    if len(lmList) != 0:
        x1, y1 = lmList[8][1:]   # Trỏ
        x2, y2 = lmList[12][1:]  # Giữa
        x4, y4 = lmList[4][1:]   # Cái
        x20, y20 = lmList[20][1:] # Út
        
        tipIds = [4, 8, 12, 16, 20]
        fingers = get_fingers_status(lmList, tipIds)

        # Chuyen che do
        if fingers == [1, 1, 1, 1, 1] and (time.time() - last_mode_switch_time) > 2:
            mode = (mode + 1) % 3 
            last_mode_switch_time = time.time()

        # === MODE 0: CHUỘT ===
        if mode == 0:
            # 1. Chup man hinh (Chu V)
            if fingers[1] == 1 and fingers[2] == 1 and fingers[3] == 0 and fingers[4] == 0:
                if time.time() - last_screenshot_time > 2:
                    filename = f"Screenshots/Screen_{datetime.now().strftime('%Y%m%d_%H%M%S')}.png"
                    pyautogui.screenshot(filename)
                    last_screenshot_time = time.time()
                    screenshot_notification_start = time.time()

            # 2. Di chuyen chuot (Full man hinh)
            elif fingers[1] == 1 and fingers[2] == 0: 
                # Mapping tu (0 -> wCam) sang (0 -> wScr)
                x3 = np.interp(x1, (0, wCam), (0, wScr))
                y3 = np.interp(y1, (0, hCam), (0, hScr))
                
                clocX = plocX + (x3 - plocX) / smoothening
                clocY = plocY + (y3 - plocY) / smoothening
                pyautogui.moveTo(clocX, clocY)
                plocX, plocY = clocX, clocY
                cv2.circle(img, (x1, y1), 15, (255, 0, 255), cv2.FILLED)

            # 3. Left Click (Cái + Trỏ)
            length_click = math.hypot(x4 - x1, y4 - y1) 
            if length_click < 40:
                cv2.circle(img, (x1, y1), 15, (0, 255, 0), cv2.FILLED)
                pyautogui.click()
            
            # 4. Right Click (Cái + Út)
            length_right_click = math.hypot(x4 - x20, y4 - y20)
            if length_right_click < 45: 
                cv2.circle(img, (x20, y20), 15, (0, 0, 255), cv2.FILLED) 
                pyautogui.rightClick()
                time.sleep(0.3) 

            # 5. Scroll
            elif fingers[2] == 1 and fingers[1] == 0: # Scroll len
                pyautogui.scroll(300)
            elif fingers[4] == 1: # Scroll xuong
                pyautogui.scroll(-300)

        # === MODE 1: VOLUME ===
        elif mode == 1:
            length = math.hypot(x4 - x1, y4 - y1)
            cv2.line(img, (x4, y4), (x1, y1), (255, 0, 255), 3)
            vol = np.interp(length, [50, 400], [minVol, maxVol]) 
            try: volume.SetMasterVolumeLevel(vol, None)
            except: pass
            cv2.putText(img, f'Vol: {int(np.interp(length, [50, 400], [0, 100]))}%', (50, 600), cv2.FONT_HERSHEY_PLAIN, 3, (255, 0, 0), 3)

        # === MODE 2: BRIGHTNESS ===
        elif mode == 2:
            brightness = np.interp(y1, [50, hCam-50], [100, 0])
            sbc.set_brightness(int(brightness))
            cv2.putText(img, f"Bright: {int(brightness)}%", (x1 + 30, y1), cv2.FONT_HERSHEY_PLAIN, 3, (255, 255, 0), 3)

    # Thong bao Screenshot
    if time.time() - screenshot_notification_start < 1.5:
        cv2.putText(img, "SCREENSHOT SAVED!", (wCam//2 - 200, hCam//2), cv2.FONT_HERSHEY_DUPLEX, 2, (0, 255, 255), 3)

    cv2.putText(img, f"MODE: {modes_name[mode]}", (50, 80), cv2.FONT_HERSHEY_PLAIN, 4, (0, 255, 0), 4)
    cv2.imshow("He thong dieu khien bang cu chi", img)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
