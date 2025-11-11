# Multi-stage build를 활용한 최적화된 Dockerfile
FROM python:3.11-slim as builder

WORKDIR /app

# 의존성 파일 복사
COPY requirements.txt .

# 의존성 설치 (wheel 빌드)
# 불필요한 의존성 제거하여 requirements.txt 업데이트
RUN pip install --no-cache-dir --user -r requirements.txt


# 실행 단계
FROM python:3.11-slim

# 보안을 위한 non-root 유저 생성
RUN groupadd -r appuser && useradd -r -g appuser -u 1000 appuser

WORKDIR /app

# builder 단계에서 설치한 패키지 복사
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local

# 애플리케이션 코드 복사
COPY --chown=appuser:appuser main.py .

# PATH 환경변수에 pip 설치 경로 추가
ENV PATH=/home/appuser/.local/bin:$PATH

# 애플리케이션 포트
EXPOSE 8000

# non-root 유저로 전환
USER appuser

# 헬스체크 설정 주석처리 (docker-compose.yml 과 중복)
# HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
#     CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# 애플리케이션 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
