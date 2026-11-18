# L.E.O.N.A. 첨부 자료

[프로젝트 README](../README.md) · [제작·시험 사진](../assets/photos/README.md)

## 설계서·보고서

| 파일 | 확인할 내용 |
| --- | --- |
| [2026 한이음 드림업 수행계획서 · PDF](haniium-project-plan.pdf) | 외벽 청소 드론을 대상으로 작성한 초기 구상 |
| [한이음 제작설계서 · PPTX](haniium-design-specification.pptx) | 하드웨어와 소프트웨어 제작 설계 |
| [한이음 개발보고서 고화질 최종본 v2 · PDF](haniium-development-report-v2.pdf) | 태양광 패널 청소 시제품의 개발과 통합 시험 |
| [한이음 개발보고서 기체 설계 작성본 · DOCX](haniium-hardware-design-draft.docx) | 기체 설계 파트 작성 자료 |
| [항공드론 종합설계경진대회 최종보고서 본문 · PDF](aircraft-drone-final-report.pdf) | 목표, 구현, 시험 결과 및 작품 사진. 팀원 연락처가 실린 부록을 제외한 23쪽. [Drive 원본](https://drive.google.com/file/d/17nlW2NVn1YP9OBiPutygKmgqJvCISR5F/view) |
| [종합설계경진대회 하드웨어 반영본 · Drive 원본](https://drive.google.com/file/d/14e-_miZilEbmMfDHToiq8BBftZwFjgnf/view) | 연락처가 포함된 편집용 보고서 |
| [종합설계경진대회 최종 결과 인수인계본 · Markdown](competition-final-report-handoff.md) | 최종 결과를 정리한 팀 인수인계 자료 |

## 발표·전시 자료

- [종합설계경진대회 최종 발표자료 · PPTX](competition-presentation.pptx)
- [2026 AD페스티벌 포스터 · PPTX](ad-festival-poster.pptx)
- [L.E.O.N.A. 인포그래픽 편집본 · PPTX](leona-infographic.pptx)

## 시험 기록과 기술 문서

| 분야 | 파일 |
| --- | --- |
| GPS·거리 제어 | [GPS 고도 현장시험 기록](tests/gps-altitude-field-test.md), [결과 CSV](tests/gps-altitude-field-test-results.csv), [Pi 현장 캘리브레이션](tests/pi-field-calibration-2026-08-21.md) |
| 물 분사부 | [솔레노이드 벤치 시험](tests/solenoid-bench-test.md), [DRV8876 밸브 구동 회로 인계](tests/drv8876-valve-driver-handoff.md), [분사 순서 시험 절차](tests/spray-sequence-test-procedure.md), [분사 반력 보상 설계](tests/spray-reaction-feedforward.md) |
| 자율비행과 현장 운용 | [시스템 구성](tests/system-architecture.md), [자율 청소 아키텍처](tests/autonomous-cleaning-architecture.md), [Pi 운용 앱](tests/pi-operator-app.md), [현장 진단](tests/field-diagnostics.md) |
| 영상 인식과 통신 | [AI 데이터 처리 과정](tests/ai-data-pipeline.md), [Dirt V3 후보 평가](tests/dirt-v3-candidate-evaluation.md), [Edge GPU 연결 검증](tests/edge-gpu-connection-runbook.md) |

시험 문서의 코드 경로와 명령 예시는 당시 개발 환경을 기준으로 작성했습니다. 일부 파일에는 내부 IP 주소와 로컬 계정명에 가림 처리를 했습니다.
