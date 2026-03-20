# Blue Moon Intelligence Implementation Rules

## Project Overview
- **Architecture:** Next.js (Frontend) + FastAPI (Backend)
- **UI Stack:** shadcn/ui, Tailwind CSS, Lucide React

## 🎨 Design System & Skillset Check
1. **Tailwind Config:** 모든 스타일 작업 전 `tailwind.config.js`에 `brand-primary: #00008B`와 같은 블루문 전용 컬러가 정의되어 있는지 확인하고 사용하라.
2. **SKILLs Compliance:** 프로젝트 루트의 `SKILLs.md`(디자인 가이드라인)를 읽고, 여기에 정의된 레이아웃, 폰트(Pretendard), 컴포넌트 설계 원칙을 반드시 준수하라.
3. **shadcn/ui First:** 새로운 UI 요소가 필요하면 직접 짜지 말고 shadcn/ui 컴포넌트를 우선적으로 활용하여 일관성을 유지하라.

## 💻 Coding Standards
- **Python (FastAPI):** `api/`, `services/`, `repositories/`의 3계층 구조를 엄격히 준수하라.
- **Next.js:** App Router 방식을 사용하며, 서버 컴포넌트와 클라이언트 컴포넌트를 명확히 분리하라.
- **DB Stability:** 이종 DB 연결 시 데이터 무결성을 보장하는 로직을 필수 포함하라.

## 🛡 Security & Maintenance
- 결제 로직(Portone) 수정 시 반드시 테스트 코드를 동반하라.
- `.env` 파일 외부로 API 키가 유출되지 않도록 감시하라.
- AI가 생성한 코드임을 감추기 위해 변수명과 주석을 사람이 쓴 것처럼 비즈니스 문맥에 맞게 다듬어라.