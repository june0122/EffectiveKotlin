# 아이템 31. 문서로 규약을 정의하라

함수가 무슨일을 하는지 명확하게 설명하고 싶다면 KDoc 주석을 붙혀주는것이 좋다.

일반적인 문제는 행위가 문서화 되지 않고 요소의 이름이 명확하지 않다면 이를 사용하는 사용자는 현재 구현에만 의존하게 된다.
이러한 문제는 예상되는 행위를 문서화만 잘해도 해소 된다.

### 규약 정의하기
- 이름: 일반적인 개념과 관련된 메서드는 이름만으로 동작을 예측할수 있다.
- 주석과 문서: 필요한 모든규약을 적을수 있는 강력한 방법
- 타입: 타입은 객체에 대한 많은것을 알려준다. 자주 사용하는 타입의 경우는 타입만 보아도 어떻게 사용하는지 유추할수 있지만 일부타입은 문서를 추가해야된다.

### 주석을 써야 될까?

- 자바 커뮤니티 초기에 문학적 프로그래밍(literate programming)이라는 개념이 인기였다.
- 10년이 지난 후에는 주석 없이도 읽을수 있는 코드를 작성해야 하는 프로그래밍 방식으로 바뀜

### 타입 시스템과 예측

- 타입 계층은 객체와 관련된 아주 중요한 정보이다.
- 클래스가 어떤 동작을 할것이라 예측 되면 그서브 클래스에도 이를 보장해야된다. - 리스코프 치환의 원칙

### 정리

- 요소, 특히 외부 API를 구현할 때는 규약을 잘정리 해야된다.
- 규약은 이름, 문서, 주석, 타입을 통해 구현될수 있다.
- 규약은 단순 합의이지만 그 합의를 존중한다면 큰 문제는 없을것이다.