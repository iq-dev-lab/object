<div align="center">
  <img src="images/cover.png" alt="오브젝트 - 코드로 이해하는 객체지향 설계" width="400" />
</div>

# 📖 오브젝트 - 코드로 이해하는 객체지향 설계

> AI와 함께 정리한 객체지향 설계 학습 저장소

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-21%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Chapters](https://img.shields.io/badge/Chapters-15-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![Appendix](https://img.shields.io/badge/Appendix-3-green?style=flat-square&logo=bookstack&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

---

## 📖 소개

이 저장소는 조영호 님의 "오브젝트: 코드로 이해하는 객체지향 설계" 책을 학습하면서 작성한 정리 노트입니다.

- 🤖 AI를 활용한 심층적인 개념 설명
- 💻 단계별 리팩토링 과정 분석
- 🔍 "왜?"에 대한 깊이 있는 탐구

> *"이론이 먼저가 아니라 코드가 먼저다"*

---

## 🎯 이 책에 대하여

"오브젝트"는 객체지향 설계의 이론과 실무를 연결하는 가교 역할을 하는 책입니다.

단순히 디자인 패턴을 암기하거나 원칙을 나열하는 것이 아니라, **왜 그런 설계를 해야 하는지**, **어떻게 변경 가능한 코드를 만들 수 있는지**를 구체적인 예제 코드를 통해 보여줍니다.

### ✨ 학습 목표

- 객체지향 설계의 핵심 개념 이해: **역할, 책임, 협력**
- **변경에 유연한** 설계 만들기
- 설계 트레이드오프를 통한 **실용적 판단력** 기르기
- 코드 리팩토링을 통한 설계 개선 경험

---

## 📚 목차

> 💡 **각 챕터를 클릭하면 상세한 학습 문서로 이동합니다**

|                    Chapter                     | 주제 | 핵심 키워드 |
|:----------------------------------------------:|------|------------|
|    **[01. 객체, 설계](./chapter01/README.md)**     | 티켓 판매 애플리케이션 | 의존성, 결합도, 응집도, 캡슐화 |
|  **[02. 객체지향 프로그래밍](./chapter02/README.md)**   | 영화 예매 시스템 | 협력, 메시지, 다형성 |
|  **[03. 역할, 책임, 협력](./chapter03/README.md)**   | 객체지향 설계의 핵심 | 책임 주도 설계 |
| **[04. 설계 품질과 트레이드오프](./chapter04/README.md)** | 영화 예매 시스템 개선 | 데이터 중심 vs 책임 중심 설계 |
|    **[05. 책임 할당하기](./chapter05/README.md)**    | 책임 할당 원칙 | GRASP 패턴 |
|  **[06. 메시지와 인터페이스](./chapter06/README.md)**   | 인터페이스 설계 | 퍼블릭 인터페이스 설계 |
|     **[07. 객체 분해](./chapter07/README.md)**     | 추상화 기법 | 프로시저 추상화, 데이터 추상화 |
|   **[08. 의존성 관리하기](./chapter08/README.md)**    | 의존성 다루기 | 의존성 역전 원칙 |
|    **[09. 유연한 설계](./chapter09/README.md)**     | 유연성 확보하기 | 개방-폐쇄 원칙 |
|  **[10. 상속과 코드 재사용](./chapter10/README.md)**   | 재사용 방법 비교 | 합성 vs 상속 |
|  **[11. 합성과 유연한 설계](./chapter11/README.md)**   | 합성 활용하기 | 믹스인, 인터페이스 |
|      **[12. 다형성](./chapter12/README.md)**      | 다형성의 종류 | 상속, 오버로딩, 제네릭 |
| **[13. 서브클래싱과 서브타이핑](./chapter13/README.md)**  | 상속의 용도 | 리스코프 치환 원칙 |
|   **[14. 일관성 있는 협력](./chapter14/README.md)**   | 설계 일관성 | 협력 패턴 |
|  **[15. 디자인 패턴과 프레임워크](./chapter15/README.md)**  | 패턴과 프레임워크 | GoF 패턴, 프레임워크 |

### 📘 부록

|                    Appendix                    | 주제 | 핵심 키워드 |
|:----------------------------------------------:|------|------------|
|  **[A. 계약에 의한 설계](./appendixA/README.md)**   | 협력의 명시적 문서화 | 사전조건, 사후조건, 불변식, 공변성 |
|  **[B. 타입 계층의 구현](./appendixB/README.md)**   | 다양한 타입 구현 방법 | 인터페이스, 추상 클래스, 덕 타이핑, 믹스인 |
| **[C. 동적인 협력, 정적인 코드](./appendixC/README.md)** | 모델과 코드의 관계 | 동적 모델, 정적 모델, 도메인 모델, TYPE OBJECT |

---

## 🎯 학습 방법

```
📖 Read → 🤖 AI Analysis → 💭 Deep Dive → 💻 Practice → 📝 Document
```

1. **개념 학습**: 각 Chapter의 `README.md` 읽기
2. **코드 분석**: 리팩토링 전/후 코드 비교
3. **AI 대화**: 궁금한 점을 AI와 대화하며 해소
4. **실습**: 코드를 직접 수정하고 실험
5. **문서화**: 이해한 내용을 자신의 언어로 정리

### 📖 각 챕터 구성

각 챕터 문서는 다음과 같은 구조로 구성됩니다:

| 섹션 | 설명 |
|------|------|
| 📌 **핵심 개념** | 챕터의 주요 학습 내용 |
| 🎯 **학습 목표** | 이 챕터를 통해 얻을 수 있는 것 |
| 💻 **코드 분석** | 리팩토링 전/후 비교 및 설계 개선 과정 |
| 🤔 **깊이 파기** | AI와 대화하며 발견한 통찰 |
| ✨ **핵심 정리** | 실전에 적용할 수 있는 원칙 |

---

## 💻 시작하기

### 📋 필요 사항
- **Java 21** 이상
- **IntelliJ IDEA** (권장) 또는 다른 Java IDE

### 1️⃣ Repository 클론
```bash
git clone https://github.com/dev-book-lab/object.git
cd object
```

### 2️⃣ 학습 방법

1. 관심 있는 챕터의 `README.md`를 읽으며 개념 학습
2. 원본 코드 저장소에서 해당 챕터의 예제 코드 확인
3. 코드를 직접 수정하고 실험하며 이해도 향상
4. 궁금한 점은 AI와 대화하며 깊이 있게 탐구

---

## 📝 작성 방식

- ✅ 책 내용 + AI를 통한 심층 분석
- ✅ 실무 관점의 추가 설명과 예제
- ✅ "왜?"에 대한 질문과 답변
- ✅ 단계별 리팩토링 과정 상세 분석
- ✅ 설계 트레이드오프 설명

---

## 🤝 기여하기

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 🙏 Reference

- **원서**: [오브젝트: 코드로 이해하는 객체지향 설계](http://www.yes24.com/Product/Goods/74219491) - 조영호 저, 위키북스
- **원서 코드**: [eternity-oop/object](https://github.com/eternity-oop/object)
- **관련 서적**: 『객체지향의 사실과 오해』 - 조영호 저

---

## ✨ Author

AI와 대화하며 기술을 깊이 이해하는 개발자의 학습 기록

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ and 🤖

**"설계는 트레이드오프의 산물이다"**

</div>
