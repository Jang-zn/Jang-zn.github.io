---
title: "[학습] Fake Repository 테스트 패턴과 추상화"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 10:00:00 +0900
categories: [학습, Backend]
tags: [학습, 테스트, Fake, Repository, 추상화, Clean Architecture]
difficulty: ⭐5
render_with_liquid: false
---

## 같은 CRUD를 9번 복사했다

TasteNote BE는 Clean Architecture라서 서비스 테스트에 DB가 필요 없다. 포트(인터페이스)에 대한 Fake 구현체를 만들어서 메모리에서 테스트한다. 문제는 Fake Repository가 9개인데, 전부 `save`, `findById`, `findAll`, `deleteById`를 똑같이 구현하고 있었다.

```java
// FakeRecipeRepository
private final Map<Long, Recipe> store = new HashMap<>();
private Long sequence = 1L;

public Recipe save(Recipe recipe) {
    if (recipe.getId() == null) recipe.setId(sequence++);
    store.put(recipe.getId(), recipe);
    return recipe;
}
public Optional<Recipe> findById(Long id) { return Optional.ofNullable(store.get(id)); }
// ... 나머지 CRUD
```

이게 9개 파일에 각각 45줄씩. 400줄 넘는 보일러플레이트.

## AbstractFakeRepository 추출

```java
public abstract class AbstractFakeRepository<T, ID> {
    protected final Map<ID, T> store = new HashMap<>();
    private long sequence = 1L;

    protected abstract ID getId(T entity);
    protected abstract void setId(T entity, long id);

    public T save(T entity) {
        if (getId(entity) == null) setId(entity, sequence++);
        store.put(getId(entity), entity);
        return entity;
    }

    public Optional<T> findById(ID id) {
        return Optional.ofNullable(store.get(id));
    }

    public List<T> findAll() {
        return new ArrayList<>(store.values());
    }

    public void deleteById(ID id) {
        store.remove(id);
    }
}
```

### 왜 abstract getId()인가

엔티티들이 공통 `Identifiable` 인터페이스를 구현하지 않는다. `Recipe.getId()`, `Purchase.getPurchaseId()` — getter 이름이 다르다. 추상 메서드로 각 Fake가 직접 구현하게 했다.

```java
public class FakeRecipeRepository extends AbstractFakeRepository<Recipe, Long>
    implements RecipeRepository {

    @Override protected Long getId(Recipe r) { return r.getId(); }
    @Override protected void setId(Recipe r, long id) { r.setId(id); }

    // 커스텀 메서드만 구현
    @Override
    public List<Recipe> findByUserId(Long userId) {
        return store.values().stream()
            .filter(r -> r.getUserId().equals(userId))
            .collect(Collectors.toList());
    }
}
```

CRUD 보일러플레이트가 사라지고 커스텀 메서드만 남는다. 파일당 약 40% 코드 감소.

## 대안 검토

| 방식 | 장점 | 단점 |
|------|------|------|
| AbstractFakeRepository | 타입 안전, 커스텀 가능 | abstract 메서드 구현 필요 |
| Identifiable 인터페이스 | abstract 불필요 | 모든 엔티티 수정 필요 |
| Mockito mock | 구현 없이 stub | 행위 검증만, 상태 검증 어려움 |

Mockito mock은 "이 메서드가 호출됐는가"를 검증하지, "데이터가 맞게 저장됐는가"를 검증하기 어렵다. Fake는 실제 in-memory 저장소이므로 상태 검증이 가능하다. 작년 학습 과정에서 배운 "Fake > Mock" 원칙을 그대로 적용.

## 프로젝트 적용

9개 Fake Repository 전부 `AbstractFakeRepository`를 상속하도록 리팩터링. 보일러플레이트 약 360줄 제거.

```
FakeRecipeRepository       → AbstractFakeRepository<Recipe, Long>
FakePurchaseRepository     → AbstractFakeRepository<Purchase, Long>
FakePantryItemRepository   → AbstractFakeRepository<PantryItem, Long>
FakeCookingLogRepository   → AbstractFakeRepository<CookingLog, Long>
... 5개 더
```
