# QueryDsl MySQL Join Case Sum GroupBy

데이터를 특정 기준에 따른 카운트 합계를 구할 때 case 구문을 잘 사용하면 여러 번의 쿼리를 날려야 할 상황에서도 한 번의 쿼리로 처리할 수 있다.

예를 들어 심사(review) 업무를 담당하는 심사자(reviewer)의 심사 실적을 구해야하는 요구사항이 있다고 하자.

심사자는 심사 대상을 승인(Approved)하거나 기각(Rejectred) 할 수 있다.

심사 대상은 zzz라고 부르며, 일반 zzz가 있고, 프리미엄 zzz가 있다.

심사 대상이 승인되면 출시(Published)되고, 기각되면 기각(Rejected)되고, 심사 과정에 문제가 있는채로 승인 됐다면 출시 이후에도 사용금지(Forbidden) 될 수 있으며 사용금지 된 것은 심사 실적에 포함하지 않는다.

특정 기간(날짜 기준)에 특정 심사자가 수행한 심사 건수를 Approved, Rejected, Forbidden 별로 구하되 zzz가 일반일 때와 프리미엄일 때를 구분해서 구해야 한다면 어떻게 해야할까?

## case 사용하지 않을 때

쉽게 생각나는 것은 특정 기간, 특정 심사자에 대해
- 심사 대상이 일반일 때
  - Approved인 건수를 구하고, 
  - Rejected인 건수를 구하고,
  - Forbidden인 건수를 구하고,
- 심사 대상이 프리미엄일 때
  - Approved인 건수를 구하고, 
  - Rejected인 건수를 구하고,
  - Forbidden인 건수를 구한다.

이렇게 구하려면 대략 다음과 같은 쿼리를 파라미터를 바꿔가면서 6회 수행해야 한다.

```sql
select date_format(r.end_time, '%Y-%m-%d') as review_date,
       r.count(1)
from review r
inner join zzz z on r.zzz_oid = z.oid
where r.reviewer_uid = '특정 심사자 ID'
  and z.is_premium = 'N'  # 또는 true
  and r.status = 'APPROVED'  # 또는 'REJECTED'
  and z.status in ('PUBLISHED', 'REJECTED')  # 또는 z.status = 'FORBIDDEN'
  and r.end_time between '기간 시작일' and '기간 종료일'
group by review_date
order by review_date asc
```

물론 이렇게 개별적으로 쿼리를 6회 날리는 대신에 1회의 쿼리로 특정 기간, 특정 심사자의 심사 데이터를 모두 응용단으로 가져와서 응용단에서 집계할 수도 있다.  
for 루프를 사용한다면 대략 아래와 같을 것이다.

```kotlin
var approvedNormal = 0L
var rejectedNormal = 0L
var forbiddenNormal = 0L

var approvedPremium = 0L
var rejectedPremium = 0L
var forbiddenPremium = 0L

val reviews = reviewRepository.getReviews(기간, reviewerUid)
for (r in reviews) {
    if (r.status == Status.APPROVED && z.is_premium == false) approvedNormal++
    if (r.status == Status.REJECTED && z.is_premium == false) rejectedNormal++
    if (z.status == Status.FORBIDDEN && z.is_premium == false) forbiddenNormal++
    
    if (r.status == Status.APPROVED && z.is_premium == true) approvedPremium++
    if (r.status == Status.REJECTED && z.is_premium == true) rejectedPremium++
    if (z.status == Status.FORBIDDEN && z.is_premium == true) forbiddenPremium++
}

// 이하 생략
```

특정 기간, 특정 심사자의 데이터를 가져올 때 db 내에서 루프가 한 번 돌고, 응용 단에서도 루프가 최소한 한 번 돌아야 하므로 최소 총 2회의 루프가 돈다.  
굉장히 복잡한 집계가 필요하다면 루프를 두 번 돌더라도 응용 단에서 로직을 관리하는 게 나을 수도 있지만, 지금처럼 단순히 카운트 집계에서 굳이 그럴 필요가 있을까?

이럴 때는 case를 사용하는 게 더 낫다. 쿼리마저 위 for 루프와 거의 동일한 형태로 구성되므로 직관적으로 이해할 수 있다.

## case 사용할 때

case와 sum을 사용하면 한 번의 쿼리로 6개 항목을 모두 구할 수 있다.

```sql
select date_format(r.end_time, '%Y-%m-%d') as review_date,

       sum(case when r.status = 'APPROVED' and z.is_premium = false then 1 else 0 end) as approved_normal,
       sum(case when r.status = 'REJECTED' and z.is_premium = false then 1 else 0 end) as rejected_normal,
       sum(case when z.status = 'FORBIDDEN' and z.is_premium = false then 1 else 0 end) as forbidden_normal,

       sum(case when r.status = 'APPROVED' and z.is_premium = true then 1 else 0 end) as approved_premium,
       sum(case when r.status = 'REJECTED' and z.is_premium = true then 1 else 0 end) as rejected_premium,
       sum(case when z.status = 'FORBIDDEN' and z.is_premium = true then 1 else 0 end) as forbidden_premium,
from review r
inner join zzz z on r.zzz_oid = z.oid
where r.reviewer_uid = '특정 심사자 ID'
  and r.end_time between '기간 시작일' and '기간 종료일'
group by review_date
order by review_date asc
```

## case 를 QueryDsl로 작성

살짝 생소하지만 다음과 같이 작성하면 된다.

```kotlin
fun reviewCountsByDate(
        range: Range<ZonedDateTime>,
        reviewerUid: String,
): List<CountsByDateProjection> {
    val reviewDate = formattedDate(qReview.endTime)
    return from(qReview)
        .innerJoin(qZzz).on(qReview.zzzOid.eq(qZzz.oid))
        .where(qReview.reviewerUid.eq(reviewerUid))
        .where(qReview.endTime.between(range.lowerBound.value.get(), range.upperBound.value.get()))
        .groupBy(reviewdDate)
        .orderBy(reviewdDate.asc())
        .select(
            QCountsByDateProjection(
                reviewDate,
                CaseBuilder().`when`(qReview.status.eq(Review.Status.APPROVED).and(qZzz.isPremium.eq(false))).then(1L).otherwise(0L).sum(),
                CaseBuilder().`when`(qReview.status.eq(Review.Status.REJECTED).and(qZzz.isPremium.eq(false))).then(1L).otherwise(0L).sum(),
                CaseBuilder().`when`(qZzz.status.eq(Zzz.Status.FORBIDDEN).and(qZzz.isPremium.eq(false))).then(1L).otherwise(0L).sum(),

                CaseBuilder().`when`(qReview.status.eq(Review.Status.APPROVED).and(qZzz.isPremium.eq(true))).then(1L).otherwise(0L).sum(),
                CaseBuilder().`when`(qReview.status.eq(Review.Status.REJECTED).and(qZzz.isPremium.eq(true))).then(1L).otherwise(0L).sum(),
                CaseBuilder().`when`(qZzz.status.eq(Zzz.Status.FORBIDDEN).and(qZzz.isPremium.eq(true))).then(1L).otherwise(0L).sum(),
            )
        )
        .fetch()
}
    
data class CountsByDateProjection @QueryProjection constructor(
    val date: String,

    val approvedNormal: Long,
    val rejectedNormal: Long,
    val forbiddentNormal: Long,

    val approvedPremium: Long,
    val rejectedPremium: Long,
    val forbiddenPremium: Long,
)
```


