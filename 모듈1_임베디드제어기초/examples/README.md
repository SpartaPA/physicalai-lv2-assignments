# 모듈 1 과제용 P 제어 예제

[실습 준비](../실습준비.md)를 먼저 확인하세요.

[opencr_position_p.ino](opencr_position_p/opencr_position_p.ino)를 사용합니다. 이번 과제는 P 제어 실행 두 번만 비교하며 PI·PID 구현과 튜닝을 요구하지 않습니다.

진행자가 확인한 설정 예시: `s 0.3 15 30`으로 실행 A, 정지 및 자세·이동 여유 확인 후 `s 0.6 15 30`으로 실행 B. `x`로 정지합니다. 각 실행은 현재 위치를 0도로 잡습니다. `max`는 사용하지 않습니다.

명령은 `s Kp 속도상한(도/초) 상대목표각(도)` 형식입니다. 실측 로그의 `target_deg`, `position_deg`, `error_deg`, `u_deg_s`, `speed_deg_s`, `t_s`를 사용합니다.
