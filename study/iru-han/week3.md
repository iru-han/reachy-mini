리치미니 모터 테스트
```
import time
import numpy as np
from reachy_mini import ReachyMini
from reachy_mini.utils import create_head_pose

def run_comprehensive_test():
    with ReachyMini() as mini:
        print("=== [1] 보간 방법(Interpolation) 비교 테스트 ===")
        # linear: 직선적, minjerk: 부드러움, cartoon: 튕기는 느낌, ease: 서서히 가속/감속
        methods = ["linear", "minjerk", "cartoon", "ease_in_out"]

        for method in methods:
            print(f"현재 방법: {method} - 움직임을 관찰하세요.")
            # 목표 지점으로 이동
            mini.goto_target(
                head=create_head_pose(y=10, mm=True),
                duration=1.5,
                method=method
            )
            time.sleep(0.5)
            # 원점으로 복귀
            mini.goto_target(head=create_head_pose(), duration=1.5, method=method)
            time.sleep(0.5)

        input("\n엔터를 누르면 [2] 실시간 제어 테스트를 시작합니다...")

        print("\n=== [2] 실시간 동작 제어 (Sine Wave) ===")
        print("10초 동안 부드럽게 좌우(Yaw)로 흔듭니다.")
        mini.goto_target(head=create_head_pose(), duration=1.0)
        start = time.time()

        while True:
            t = time.time() - start
            if t > 10: break

            # y = A * sin(2 * pi * f * t)
            # 10mm 진폭으로 0.5Hz 주파수(2초에 한 번 왕복) 운동
            y_pos = 10 * np.sin(2 * np.pi * 0.5 * t)
            
            # set_target은 goto_target과 달리 대기하지 않고 즉시 명령을 보냅니다.
            mini.set_target(head=create_head_pose(y=y_pos, mm=True))
            time.sleep(0.01) # 100Hz 주기로 업데이트

        mini.enable_motors()

        input("\n엔터를 누르면 [3] 모터 제어(컴플라이언스) 테스트를 시작합니다...")
        mini.disable_motors()
        mini.compliant = True
        
        print("\n=== [3] 모터 제어 테스트 (중력 보상) ===")
        print("지금부터 5초간 로봇 머리가 '흐물흐물'해집니다. 손으로 직접 움직여보세요!")
        
        for i in range(10, 0, -1):
            print(f"{i}초 남음...")
            time.sleep(1)
            
        print("모터를 다시 잠급니다(stiff).")
        mini.enable_motors()
        mini.goto_target(head=create_head_pose(), duration=1.0)

        input("\n엔터를 누르면 [4] 안전 범위 테스트를 시작합니다...")

        print("\n=== [4] 안전 범위 테스트 ===")
        # Roll 제한은 보통 -40~40도입니다. -50도를 입력해봅니다.
        print("제한 범위(-40도)를 벗어나는 -50도 회전을 시도합니다.")
        excessive_pose = create_head_pose(roll=-50, degrees=True)
        mini.goto_target(head=excessive_pose, duration=1.5)

        # 실제 도달한 위치 확인 (로봇이 스스로 제한한 값을 확인 가능)
        current_pose = mini.get_current_head_pose()
        print(f"명령값: -50도 | 실제 도달 포즈: {current_pose}")

        print("\n모든 테스트가 완료되었습니다. 초기 위치로 복귀합니다.")
        mini.goto_target(head=create_head_pose(), antennas=[0, 0], duration=2.0)

if __name__ == "__main__":
    run_comprehensive_test()
```

