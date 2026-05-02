```
pip install pyaudio sounddevice numpy scipy
```

tts 관련 설치
```
pip install pyttsx3 gTTS SpeechRecognition openai-whisper
```

마이크 장치 확인 (check_audio.py)
```
import sounddevice as sd

print("--- 사용 가능한 오디오 장치 목록 ---")
print(sd.query_devices())

print("\n--- 기본 설정 장치 ---")
print(f"기본 입력: {sd.query_devices(kind='input')['name']}")
print(f"기본 출력: {sd.query_devices(kind='output')['name']}")
```
-> python check_audio.py 로 실행
<img width="695" height="418" alt="image" src="https://github.com/user-attachments/assets/282925af-70d7-45d4-ab06-f57075c6ff6e" />

마이크 입력 녹음 (mic_record.py)
```
import sounddevice as sd
import numpy as np
from scipy.io.wavfile import write

# 녹음 설정
SAMPLE_RATE = 16000  # Reachy Mini 마이크 최대 샘플레이트
DURATION = 5  # 녹음 시간 (초)
CHANNELS = 1  # 모노 녹음

print(f"{DURATION}초 동안 녹음을 시작합니다...")

# 녹음 수행
audio_data = sd.rec(
    int(DURATION * SAMPLE_RATE),
    samplerate=SAMPLE_RATE,
    channels=CHANNELS,
    dtype='int16'
)

# 녹음이 완료될 때까지 대기
sd.wait()

print("녹음이 완료되었습니다.")

# WAV 파일로 저장
output_filename = "recorded_audio.wav"
write(output_filename, SAMPLE_RATE, audio_data)
print(f"오디오가 '{output_filename}'에 저장되었습니다.")
```
<img width="386" height="61" alt="image" src="https://github.com/user-attachments/assets/dd8d0955-0878-4ae3-b219-051bf4a0b611" />


실시간 오디오 스트림 처리(audio_stream.py)
```
import sounddevice as sd
import numpy as np

SAMPLE_RATE = 16000
BLOCK_SIZE = 1024  # 한 번에 처리할 샘플 수

def audio_callback(indata, frames, time, status):
    """실시간 오디오 입력 콜백 함수"""
    if status:
        print(f"상태: {status}")
    
    # 입력 오디오의 볼륨 레벨 계산 (RMS)
    volume_norm = np.linalg.norm(indata) * 10
    
    # 볼륨 레벨을 시각적으로 표시
    bar = '█' * int(volume_norm)
    print(f"\r볼륨: {bar:<50}", end='', flush=True)
    print(f"\volume_norm: {volume_norm}", flush=True)

print("실시간 오디오 모니터링을 시작합니다. Ctrl+C로 종료하세요.")

try:
    with sd.InputStream(
        samplerate=SAMPLE_RATE,
        blocksize=BLOCK_SIZE,
        channels=1,
        callback=audio_callback
    ):
        while True:
            sd.sleep(100)
except KeyboardInterrupt:
    print("\n오디오 모니터링을 종료합니다.")
```
<img width="464" height="50" alt="image" src="https://github.com/user-attachments/assets/964218a7-9aca-4e6a-a02e-e5d313cfb40a" />

WAV 재생
```
import sounddevice as sd
from scipy.io.wavfile import read

# WAV 파일 읽기
sample_rate, audio_data = read("recorded_audio.wav")

print(f"재생 중... (샘플레이트: {sample_rate} Hz)")

# 오디오 재생
sd.play(audio_data, sample_rate)

# 재생이 완료될 때까지 대기
sd.wait()

print("재생이 완료되었습니다.")
```
<img width="318" height="50" alt="image" src="https://github.com/user-attachments/assets/ddcc6454-1ab1-400c-8cf8-364b214ce76b" />

사인파 톤 생성 및 재생
1. 소리는 '파동'이다 (Sine 함수)<br/>
가장 먼저 이해해야 할 것은 np.sin()입니다. 소리는 공기의 떨림이고, 그 떨림을 그래프로 그리면 위아래로 부드럽게 움직이는 사인(Sine) 곡선이 됩니다.<br/>
수식의 역할: 사인 함수는 어떤 값을 넣든 -1에서 1 사이를 반복해서 왔다 갔다 하는 숫자를 만들어냅니다.<br/>
비유: 시계추가 왼쪽 끝(-1)에서 오른쪽 끝(1)으로 왔다 갔다 하는 운동을 상상해 보세요. 그 움직임이 바로 소리의 기본인 '사인파'가 됩니다.<br/>

2. 수식의 구성 요소 파헤치기
<img width="193" height="33" alt="image" src="https://github.com/user-attachments/assets/ee224b57-581a-4d5c-aeb1-fe6e0440b0a3" />

np.linspace(시작, 끝, 개수, False): 어디서부터 어디까지 몇 개의 숫자를 일정한 간격으로 만들어줘.<br/>
* 0 (시작): 녹음(또는 재생) 시작 시간인 0초를 의미합니다.<br/>
* duration (끝): 우리가 설정한 1.0초를 의미합니다.<br/>
* int(sample_rate * duration) (개수): sample_rate가 44,100이라면, 1초에 44,100개의 점을 찍겠다는 뜻. 즉, "1초 동안 총 44,100개의 시간 데이터(점)를 만들어라"는 명령. 이 점들이 모여서 소리의 해상도를 결정.<br/>

2 * np.pi(한 바퀴): 수학에서 사인 함수가 한 바퀴(위로 갔다 아래로 돌아오기)를 완전히 도는 데 필요한 값<br/>
frequency(진동수/주파수): 1초에 몇 번 떨릴 것인가<br/>
t(시간): 시간의 흐름<br/>

3. 마지막에 32767을 곱하는 이유<br/>
audio = (tone * 32767).astype(np.int16)<br/>
사인 함수가 만든 값은 -1에서 1 사이의 아주 작은 소수점 숫자들입니다. 하지만 컴퓨터의 오디오 카드(스피커)는 보통 16비트 정수라는 규격으로 소리를 읽습니다.<br/>
16비트 정수의 범위: -32768 ~ 32767<br/>
```
import sounddevice as sd
import numpy as np

SAMPLE_RATE = 44100  # CD 품질 샘플레이트
DURATION = 1.0  # 재생 시간 (초)

def generate_tone(frequency, duration, sample_rate):
    """특정 주파수의 사인파 생성"""
    t = np.linspace(0, duration, int(sample_rate * duration), False)
    tone = np.sin(2 * np.pi * frequency * t)
    # 16비트 정수로 변환
    audio = (tone * 32767).astype(np.int16)
    return audio

# 다양한 음계 재생
notes = {
    'C4': 261.63,
    'D4': 293.66,
    'E4': 329.63,
    'F4': 349.23,
    'G4': 392.00,
    'A4': 440.00,
    'B4': 493.88,
    'C5': 523.25
}

print("음계를 재생합니다...")

for note_name, frequency in notes.items():
    print(f"재생 중: {note_name} ({frequency} Hz)")
    tone = generate_tone(frequency, 0.5, SAMPLE_RATE)
    sd.play(tone, SAMPLE_RATE)
    sd.wait()

print("재생이 완료되었습니다.")
```
<img width="323" height="177" alt="image" src="https://github.com/user-attachments/assets/d20cede7-3fc9-4613-8fc7-1f4edfccd10c" />

tts
```
import pyttsx3

# TTS 엔진 초기화
engine = pyttsx3.init()

# 음성 속성 설정
engine.setProperty('rate', 150)    # 말하기 속도 (기본값: 200)
engine.setProperty('volume', 0.9)  # 볼륨 (0.0 ~ 1.0)

# 사용 가능한 음성 목록 확인
voices = engine.getProperty('voices')
for idx, voice in enumerate(voices):
    print(f"{idx}: {voice.name} - {voice.languages}")

# 한국어 음성이 있으면 선택 (시스템에 따라 다름)
# engine.setProperty('voice', voices[1].id)

# 텍스트를 음성으로 변환하여 재생
text = "안녕하세요, 저는 리치 미니입니다."
print(f"말하는 중: {text}")
engine.say(text)
engine.runAndWait()

print("TTS 재생이 완료되었습니다.")
```
<img width="509" height="81" alt="image" src="https://github.com/user-attachments/assets/95c9da41-46b4-4892-8420-ea88316a22d7" />

gtts 사용
```
from gtts import gTTS
import os

# 한국어 텍스트
text = "안녕하세요, 저는 리치 미니입니다. 만나서 반갑습니다."

# gTTS로 음성 생성 (한국어)
tts = gTTS(text=text, lang='ko', slow=False)

# MP3 파일로 저장
output_file = "greeting.mp3"
tts.save(output_file)
print(f"음성 파일이 '{output_file}'에 저장되었습니다.")

# 파일 재생 (시스템에 따라 다른 방법 사용)
# Windows
os.system(f"start {output_file}")
# macOS: os.system(f"afplay {output_file}")
# Linux: os.system(f"mpg123 {output_file}")
```
<img width="353" height="16" alt="image" src="https://github.com/user-attachments/assets/2a7aa19c-bacb-4179-9f18-b7f309f72b8d" />

speech recognition 설치
```
pip install SpeechRecognition
```

음성인식
```
import speech_recognition as sr

# 음성 인식기 초기화
recognizer = sr.Recognizer()

# 마이크에서 음성 입력 받기
with sr.Microphone() as source:
    print("주변 소음을 분석 중...")
    recognizer.adjust_for_ambient_noise(source, duration=1)
    
    print("말씀하세요...")
    audio = recognizer.listen(source, timeout=5, phrase_time_limit=10)
    
print("인식 중...")

try:
    # Google Speech Recognition API 사용 (무료, 인터넷 필요)
    text = recognizer.recognize_google(audio, language='ko-KR')
    print(f"인식 결과: {text}")
except sr.UnknownValueError:
    print("음성을 인식할 수 없습니다.")
except sr.RequestError as e:
    print(f"Google Speech Recognition 서비스 오류: {e}")
```
<img width="371" height="82" alt="image" src="https://github.com/user-attachments/assets/dfe1e9f2-d365-4970-91b0-cfe951d56ec7" />

ffmpg 설치 (power shell에서)
```
winget install ffmpeg
```

터미널을 껐다 킨 후, 아무 폴더에서나 아래 명령어 치기
```
ffmpeg -version
```
<img width="599" height="177" alt="image" src="https://github.com/user-attachments/assets/7a844b21-19fe-47db-a0bd-9ddd8988976b" />


Whisper를 이용한 고급 음성 인식
```
import whisper
import sounddevice as sd
import numpy as np
from scipy.io.wavfile import write
import tempfile
import os

# Whisper 모델 로드 (tiny, base, small, medium, large 중 선택)
# 작은 모델일수록 빠르지만 정확도가 낮음
print("Whisper 모델을 로드 중...")
model = whisper.load_model("large")

SAMPLE_RATE = 16000
DURATION = 5

def record_audio(duration, sample_rate):
    """마이크에서 오디오 녹음"""
    print(f"{duration}초 동안 녹음합니다...")
    audio = sd.rec(
        int(duration * sample_rate),
        samplerate=sample_rate,
        channels=1,
        dtype='float32'
    )
    sd.wait()
    print("녹음 완료!")
    return audio.flatten()

def transcribe_audio(audio_data, sample_rate):
    """Whisper로 음성을 텍스트로 변환"""
    # 임시 WAV 파일 생성
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        temp_path = f.name
        # float32를 int16으로 변환하여 저장
        audio_int16 = (audio_data * 32767).astype(np.int16)
        write(temp_path, sample_rate, audio_int16)
    
    try:
        # Whisper로 음성 인식
        result = model.transcribe(temp_path, language='ko')
        return result['text']
    finally:
        # 임시 파일 삭제
        os.unlink(temp_path)

# 메인 실행
print("\n음성 인식을 시작합니다.")
audio = record_audio(DURATION, SAMPLE_RATE)
print(f"\naudio: {audio}")
text = transcribe_audio(audio, SAMPLE_RATE)
print(f"\n인식 결과: {text}")
```
<img width="675" height="431" alt="image" src="https://github.com/user-attachments/assets/646fe981-9b55-4144-8124-efe67a8ea970" />

음성 대화 시스템<br/>
```
import speech_recognition as sr
import pyttsx3
import time

class VoiceAssistant:
    def __init__(self):
        # 음성 인식기 초기화
        self.recognizer = sr.Recognizer()
        
        # TTS 엔진 초기화
        self.tts_engine = pyttsx3.init()
        self.tts_engine.setProperty('rate', 150)
        
        # 명령어 매핑
        self.commands = {
            '안녕': self.greet,
            '시간': self.tell_time,
            '종료': self.shutdown,
        }
        
        self.running = True
    
    def speak(self, text):
        """텍스트를 음성으로 출력"""
        print(f"[리치 미니]: {text}")
        self.tts_engine.say(text)
        self.tts_engine.runAndWait()
    
    def listen(self):
        """마이크에서 음성 입력을 받아 텍스트로 변환"""
        with sr.Microphone() as source:
            print("\n듣고 있습니다...")
            self.recognizer.adjust_for_ambient_noise(source, duration=0.5)
            
            try:
                audio = self.recognizer.listen(source, timeout=5, phrase_time_limit=10)
                text = self.recognizer.recognize_google(audio, language='ko-KR')
                print(f"[사용자]: {text}")
                return text
            except sr.WaitTimeoutError:
                return None
            except sr.UnknownValueError:
                return None
            except sr.RequestError:
                self.speak("음성 인식 서비스에 연결할 수 없습니다.")
                return None
    
    def greet(self):
        """인사 응답"""
        self.speak("안녕하세요! 저는 리치 미니입니다. 무엇을 도와드릴까요?")
    
    def tell_time(self):
        """현재 시간 알려주기"""
        current_time = time.strftime("%H시 %M분")
        self.speak(f"현재 시간은 {current_time}입니다.")
    
    def shutdown(self):
        """시스템 종료"""
        self.speak("안녕히 가세요!")
        self.running = False
    
    def process_command(self, text):
        """명령어 처리"""
        if text is None:
            return
        
        # 등록된 명령어 확인
        for keyword, action in self.commands.items():
            if keyword in text:
                action()
                return
        
        # 알 수 없는 명령어
        self.speak(f"'{text}'를 이해하지 못했습니다. 다시 말씀해 주세요.")
    
    def run(self):
        """메인 루프 실행"""
        self.speak("음성 대화 시스템을 시작합니다.")
        
        while self.running:
            text = self.listen()
            self.process_command(text)
        
        print("음성 대화 시스템을 종료합니다.")


if __name__ == "__main__":
    assistant = VoiceAssistant()
    assistant.run()
```
<img width="543" height="437" alt="image" src="https://github.com/user-attachments/assets/ae503ce3-2908-4b29-b963-20abd5bc3d39" />

음성제어
```
import speech_recognition as sr
import pyttsx3
from reachy_mini import ReachyMini

class ReachyVoiceControl:
    def __init__(self):
        # Reachy Mini 연결
        self.reachy = ReachyMini(localhost_only=True)
        self.reachy.enable_motors()
        
        # 음성 시스템 초기화
        self.recognizer = sr.Recognizer()
        self.tts = pyttsx3.init()
        
        # 명령어 등록
        self.commands = {
            '위': lambda: self.look_direction(0.5, 0, 0.5),
            '아래': lambda: self.look_direction(0.5, 0, 0.2),
            '왼쪽': lambda: self.look_direction(0.5, 0.3, 0.35),
            '오른쪽': lambda: self.look_direction(0.5, -0.3, 0.35),
            '앞': lambda: self.look_direction(0.5, 0, 0.35),
        }
    
    def speak(self, text):
        """음성 출력"""
        print(f"[리치]: {text}")
        self.tts.say(text)
        self.tts.runAndWait()
    
    def listen(self):
        """음성 입력"""
        with sr.Microphone() as source:
            print("명령을 기다리는 중...")
            self.recognizer.adjust_for_ambient_noise(source, duration=0.5)
            try:
                audio = self.recognizer.listen(source, timeout=5)
                return self.recognizer.recognize_google(audio, language='ko-KR')
            except:
                return None
    
    def look_direction(self, x, y, z):
        """지정된 방향 바라보기"""
        self.reachy.look_at_world(x=x, y=y, z=z, duration=1.0)
    
    def run(self):
        """메인 루프"""
        self.speak("음성 제어 모드를 시작합니다.")
        
        while True:
            text = self.listen()
            if text is None:
                continue
            
            print(f"인식: {text}")
            
            if '종료' in text:
                self.speak("음성 제어를 종료합니다.")
                break
            
            # 방향 명령 처리
            for keyword, action in self.commands.items():
                if keyword in text:
                    self.speak(f"{keyword}을 바라봅니다.")
                    action()
                    break
            else:
                self.speak("알 수 없는 명령입니다.")


if __name__ == "__main__":
    controller = ReachyVoiceControl()
    controller.run()
```
<img width="932" height="665" alt="image" src="https://github.com/user-attachments/assets/5b37884a-f2d5-42e7-946c-25543c3ad0a0" />
