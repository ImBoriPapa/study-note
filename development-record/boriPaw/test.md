
# 계층간 의존성 개선

## 개요
- 고수준 모듈이 저수준 모듈에 의존

## DIP (의존성 역전 원칙)
- 서술
## 변경 사항 상세
- 변경사항
## 이전 의존성 구조

```
interfaces -> application -> MemberRository

interfaces -> application -> domain -> infra
```

## DIP 적용 후 의존성 구조

![DependencyDirection](https://github.com/ImBoriPapa/study-note/assets/98242564/9a6ed75d-c2e9-4ca9-a089-8643d1f6d63c)

## 이점 및 영향

## 시각적 표현 포함
```
bad-request
└── member
    ├── command
    │   ├── interfaces
    │   ├── application
    │   │   ├── memberSignupService
    │   ├── domain
    │   │   ├── interface MemberRepository extends JpaRepository<Member, Long>
    │   │   ├── interface ImageUploader
    │   └── infra
    │       ├── other..
    ├── query
    │   ├── interfaces
    │   ├── dto
    │   └── dao..
 ```


```
bad-request
└── member
    ├── command
    │   ├── interfaces
    │   ├── application
    │   │   ├── memberSignupService
    │   ├── domain
    │   │   ├── interface MemberRepository
    │   │   ├── interface ProfileImageUploader
    │   └── infra
    │       ├── persistence
    │           ├── interface MemberJpaRepository extends JpaRepository<Member, Long>, MemberRepository
    │       ├── upload
    │           ├── ProfileImageUploaderImpl
    ├── query
    │   ├── interfaces
    │   ├── dto
    │   └── dao..
 ```



## 미래 고려 사항

## 결론
