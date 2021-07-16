# QueryDSL 불필요한 cross join 제거

아래와 같은 QueryDSL 코드를 실행하면,

```kotlin
from(qWorldVersion)
    .innerJoin(qWorld).on(qWorldVersion.`in`(qWorld.publishedVersion, qWorld.notReviewedVersion, qWorld.approvedVersion, qWorld.rejectedVersion))
    .innerJoin(qWorldCategory1).on(qWorldVersion.category1Id.eq(qWorldCategory1.id))
    .innerJoin(qWorldCategory2).on(qWorldVersion.category2Id.eq(qWorldCategory2.id))
    .innerJoin(qResourceFile).on(qWorldVersion.coverImage.oid.eq(qResourceFile.oid))
    .innerJoin(qPackageFile).on(qWorldVersion.packageFile.oid.eq(qPackageFile.oid))
.where(
    eqReviewId(target, keyword),
    containsWorldId(target, keyword),
    containsCreatorUid(target, keyword),
    containsReviewerId(target, keyword),
)
.select(
    QWorldSearchPrj(
        qWorld.oid,
        qWorld.worldId,
        qWorldVersion.name,
        qWorldVersion.description,
        qWorldVersion.coverImage.filePath,  // 여기 주목
        qWorldVersion.tags,
        qWorld.publishedCount.stringValue(),
        qWorldVersion.items.size(),
        qWorldVersion.packageFile.version,  // 여기 주목
        qPackageFile.version,
        qWorldCategory1.name,
        qWorldCategory2.name,
        qWorldVersion.submittedTime,
        qWorldVersion.publishedTime,
        qWorldVersion.statusChangedTime,
        qWorld.status,
        qWorldVersion.status,
    )
)
.fetchAll()
```

아래와 깉이 불필요한 cross join 이 추가된다.

```sql
select 
  world1_.oid as col_0_0_, world1_.world_id as col_1_0_, worldversi0_.name as col_2_0_, worldversi0_.description as col_3_0_, resourcefi6_.file_path as col_4_0_, worldversi0_.tags as col_5_0_, cast(world1_.published_count as char) as col_6_0_, (select count(items8_.world_version_id) from world_item items8_ where worldversi0_.oid = items8_.world_version_id) as col_7_0_, packagefil7_.version as col_8_0_, worldcateg2_.name as col_9_0_, worldcateg3_.name as col_10_0_, worldversi0_.submitted_time as col_11_0_, worldversi0_.published_time as col_12_0_, worldversi0_.status_changed_time as col_13_0_, world1_.status as col_14_0_, worldversi0_.status as col_15_0_ 
from world_version worldversi0_ 
  inner join world world1_ on (worldversi0_.oid in (world1_.published_version_id , world1_.not_reviewed_version_id , world1_.approved_version_id , world1_.rejected_version_id)) 
  inner join world_category worldcateg2_ on (worldversi0_.category1id=worldcateg2_.id) 
  inner join world_category worldcateg3_ on (worldversi0_.category2id=worldcateg3_.id) 
  inner join resource_file resourcefi4_ on (worldversi0_.cover_image_id=resourcefi4_.oid)    // (1)
  inner join package_file packagefil5_ on (worldversi0_.package_file_id=packagefil5_.oid)    // (2)
  cross join resource_file resourcefi6_    // (3)
  cross join package_file packagefil7_     // (4)
where 
  worldversi0_.cover_image_id=resourcefi6_.oid         // (5)
  and worldversi0_.package_file_id=packagefil7_.oid    // (6)
  and (world1_.creator_id like '%6043791d%' escape '!') limit 2;
```

(1), (2) 에서 inner join 이 돼 있음에도 불구하고, (3)~(6) 에 불필요한 cross join 과 where 조건이 추가됐다.

QueryDSL 코드를 아래와 같이 `qWorldVersion.coverImage.filePath` 대신에 `qResourceFile.filePath`로,  
`qWorldVersion.packageFile.version` 대신에 `qPackageFile.version` 로 변경하면,

```kotlin
from(qWorldVersion)
    .innerJoin(qWorld).on(qWorldVersion.`in`(qWorld.publishedVersion, qWorld.notReviewedVersion, qWorld.approvedVersion, qWorld.rejectedVersion))
    .innerJoin(qWorldCategory1).on(qWorldVersion.category1Id.eq(qWorldCategory1.id))
    .innerJoin(qWorldCategory2).on(qWorldVersion.category2Id.eq(qWorldCategory2.id))
    .innerJoin(qResourceFile).on(qWorldVersion.coverImage.oid.eq(qResourceFile.oid))
    .innerJoin(qPackageFile).on(qWorldVersion.packageFile.oid.eq(qPackageFile.oid))
.where(
    eqReviewId(target, keyword),
    containsWorldId(target, keyword),
    containsCreatorUid(target, keyword),
    containsReviewerId(target, keyword),
)
.select(
    QWorldSearchPrj(
        qWorld.oid,
        qWorld.worldId,
        qWorldVersion.name,
        qWorldVersion.description,
//      qWorldVersion.coverImage.filePath,
        qResourceFile.filePath,
        qWorldVersion.tags,
        qWorld.publishedCount.stringValue(),
        qWorldVersion.items.size(),
//      qWorldVersion.packageFile.version,
        qPackageFile.version,
        qWorldCategory1.name,
        qWorldCategory2.name,
        qWorldVersion.submittedTime,
        qWorldVersion.publishedTime,
        qWorldVersion.statusChangedTime,
        qWorld.status,
        qWorldVersion.status,
    )
)
.fetchAll()
```

아래와 같이 불필요한 cross join 과 where 조건이 사라진다.

```sql
select 
  world1_.oid as col_0_0_, world1_.world_id as col_1_0_, worldversi0_.name as col_2_0_, worldversi0_.description as col_3_0_, resourcefi4_.file_path as col_4_0_, worldversi0_.tags as col_5_0_, cast(world1_.published_count as char) as col_6_0_, (select count(items6_.world_version_id) from world_item items6_ where worldversi0_.oid = items6_.world_version_id) as col_7_0_, packagefil5_.version as col_8_0_, worldcateg2_.name as col_9_0_, worldcateg3_.name as col_10_0_, worldversi0_.submitted_time as col_11_0_, worldversi0_.published_time as col_12_0_, worldversi0_.status_changed_time as col_13_0_, world1_.status as col_14_0_, worldversi0_.status as col_15_0_ 
from world_version worldversi0_ 
  inner join world world1_ on (worldversi0_.oid in (world1_.published_version_id , world1_.not_reviewed_version_id , world1_.approved_version_id , world1_.rejected_version_id)) 
  inner join world_category worldcateg2_ on (worldversi0_.category1id=worldcateg2_.id) 
  inner join world_category worldcateg3_ on (worldversi0_.category2id=worldcateg3_.id) 
  inner join resource_file resourcefi4_ on (worldversi0_.cover_image_id=resourcefi4_.oid) 
  inner join package_file packagefil5_ on (worldversi0_.package_file_id=packagefil5_.oid) 
where world1_.creator_id like '%6043791d%' escape '!' limit 2;
```

