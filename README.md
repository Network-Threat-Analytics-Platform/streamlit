# Threat Detection Dashboard

S3에 저장된 RAG 결과(JSONL)를 실시간으로 로드하여 네트워크 위협 세션을 분석·시각화하는 Streamlit 대시보드입니다.

---

## 주요 기능

- **실시간 데이터 로드**: S3 `rag_result/dt={날짜}/` 경로 하위의 JSONL 파일 전체를 30초마다 자동 갱신
- **위협 등급 분류**: 위협 점수 기반으로 상(80점+) / 중(60~80점) / 하(60점 미만) 3단계 분류
- **다양한 검색 필터**: 날짜, IP 주소, 위협 유형, 키워드(요약/권고 조치), 위협 등급 복합 필터링
- **위협 분석 보고서**: 필터링된 세션을 테이블 형태로 10개씩 페이지네이션 표시
- **세션 상세 정보**: 등급별 라디오 필터 + 20개씩 숫자 버튼 페이지네이션, expander로 세션별 상세 보기
- **Neo4j 연관 그래프**: 1-hop 이웃 노드를 카드 목록 및 pyvis 인터랙티브 그래프로 시각화

---

## 기술 스택

| 구분 | 기술 |
|------|------|
| 대시보드 | Streamlit |
| 데이터 소스 | AWS S3 (JSONL) |
| 그래프 DB | Neo4j |
| 그래프 시각화 | pyvis |
| 데이터 처리 | pandas |
| 인프라 | Docker / Docker Compose |

---

## 디렉토리 구조

```
streamlit/
├── app.py                  # 메인 대시보드 애플리케이션
├── main.py                 # 진입점
├── requirements.txt        # Python 의존성
├── pyproject.toml          # 프로젝트 메타데이터
├── Dockerfile
├── docker-compose.yml
├── config.toml             # Streamlit 서버 설정
├── .python-version         # pyenv Python 버전 고정
└── png/                    # 결과 이미지 리소스
```

---

## 환경변수 설정

프로젝트 루트에 `.env` 파일을 생성하고 아래 항목을 설정합니다.

```env
S3_BUCKET_NAME=your-bucket-name
S3_RAG_PREFIX=rag_result
KST_TZ=Asia/Seoul          # 선택사항 (기본값: Asia/Seoul)
```

> AWS 자격증명은 `~/.aws/credentials` 또는 IAM 역할(EKS 환경)로 제공합니다.

---

## 실행 방법

### 로컬 실행

```bash
pip install -r requirements.txt
streamlit run app.py
```

### Docker Compose

```bash
docker-compose up --build
```

서버는 기본적으로 `http://localhost:8501` 에서 접근 가능합니다.

---

## Streamlit 서버 설정 (`config.toml`)

```toml
[server]
headless = true
port = 8501
enableCORS = false
```

---

## 위협 점수 기준

LLM이 반환하는 raw 점수(80점 만점)를 100점 만점으로 정규화하여 등급을 부여합니다.

| 등급 | 점수 범위 | 색상 |
|------|-----------|------|
| 상 | 80점 이상 | 🔴 |
| 중 | 60점 이상 ~ 80점 미만 | 🟡 |
| 하 | 60점 미만 | 🟢 |

---

## S3 데이터 경로 규칙

```
s3://{S3_BUCKET_NAME}/{S3_RAG_PREFIX}/dt={YYYY-MM-DD}/hour={HH}_minute={MM}_rag_results.jsonl
```

각 JSONL 라인은 아래 구조를 따릅니다.

```json
{
  "uid": "세션 고유 ID",
  "inference_datetime": "추론 시각 (ISO 8601)",
  "session": { "src_ip": "...", "dest_ip": "...", "proto": "...", ... },
  "analysis": { "threat_type": "...", "summary": "...", "threat_score": 64, "recommended_action": "..." },
  "neighbors": [ { "rel_type": "CONNECTED_TO", "node_labels": ["IP"], "node_value": "...", ... } ]
}
```
