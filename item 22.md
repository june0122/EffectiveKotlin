# 아이템 22. 인터페이스는 타입을 정의하는 용도로만 사용하라

### 안티패턴: 상수 인터페이스

```kotlin
public interface PhysicalConstants {
    // 인터페이스 내부의 필드는 `static final`이 자동으로 붙는다.
    // 아보가드로 수 (1/몰)
    double AVOGADROS_NUMBER = 6.022_140_857e23;
    // 볼츠만 상수 (J/K)
    double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    // 전자 질량 (kg)
    double ELECTRON_MASS = 9.109_383_56e-31;
}

@Test
public void test() {
    System.out.println(PhysicalConstants.AVOGADROS_NUMBER);
}
```

위의 숫자 리터럴에 사용된 밑줄은 숫자를 3자리씩 끊어 더 읽기 편하게 해준다.

- 상수 인터페이스 안티패턴은 인터페이스를 잘못 사용한 예이다.
- 상수 인터페이스를 구현하는 것은 내부 구현을 클래스의 API로 노출하는 행위이다.
- 클래스가 어떤 상수 인터페이스를 사용하든 사용자에게는 아무런 의미가 없다.
- 사용자에게 혼란을 주기도 하고, 클라이언트 코드가 내부 구현에 해당하는 이 상수들에 종속되게 한다.
  - 추후에 이 상수들을 쓰지 않아도 여전히 상수 인터페이스를 구현하고 있어야 한다.

### 더 나은 방법: 상수 유틸리티 클래스

```kotlin
public static class PhysicalConstants {
    // 아보가드로 수 (1/몰)
    public static double AVOGADROS_NUMBER = 6.022_140_857e23;
    // 볼츠만 상수 (J/K)
    public static double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    // 전자 질량 (kg)
    public static double ELECTRON_MASS = 9.109_383_56e-31;
}

@Test
public void test() {
    System.out.println(PhysicalConstants.AVOGADROS_NUMBER);
}
```

연관이 깊은 특정 클래스나 인터페이스 자체에 추가하는 것이 좋다.