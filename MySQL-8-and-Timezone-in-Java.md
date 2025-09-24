# MySQL 8 and Timezone in Java

- [MySQL dev blog](https://dev.mysql.com/blog-archive/support-for-date-time-types-in-connector-j-8-0/) 내용 재구성


## 타임존 설정 방법

- MySQL 8 에서 애플리케이션 서버 연결에 사용되는 타임존은 JDBC URL 파라미터를 통해 설정한다.

>- connectionTimeZone=LOCAL|SERVER|user-defined-time-zone (previously known as ‘serverTimezone’, now with additional fixed values) defines how the server’s session time zone is to be determined by Connector/J.
>  - DB 연결에 사용되는 타임존, 기본값 SERVER
>- forceConnectionTimeZoneToSession=true|false controls whether the session time_zone variable is to be set to the value specified in ‘connectionTimeZone’.
>  - 연결에 사용되는 타임존을 세션 타임존에 강제 적용 여부, 기본값 true
>- preserveInstants=true|false turns on|off the conversion of instant values between JVM and ‘connectionTimeZone’.
>  - JVM과 connectionTimeZone 사이의 instant 값 변환 여부, 기본값 false

## 


## Connector/J and Timezone

- 다음과 같은 코드로 MySQL DB에 `2020-01-01 12:00:00`를 저장하고 조회하면 어떻게 되는지 시나리오 별로 살펴보자.

```java
// 저장 요청
Statement st = conn.createStatement();
st.executeUpdate("CREATE TABLE t1 (ts TIMESTAMP)");

PreparedStatement ps = conn.prepareStatement("INSERT INTO t1 VALUES (?)");
ps.setTimestamp(1, Timestamp.valueOf("2020-01-01 12:00:00"));
ps.executeUpdate();


// 조회 요청
ResultSet rs = st.executeQuery("SELECT * FROM t1");
Timestamp ts = rs.getTimestamp(1);
```

### Connector/J 5.1


Who | Timezone | Action | Value
--- | --- | --- | ---
DB Client | UTC+2 | 저장 요청 | `2020-01-01 12:00:00`
DB Server | UTC+1 | 내부 TIMESTAMP 저장 | `2020-01-01 11:00:00Z`에 해당하는 타임스탬프
DB Client | UTC+3 | 조회 요청 | `2020-01-01 12:00:00`
DB Client | UTC+4 | 조회 요청 | `2020-01-01 12:00:00`


```
client offset: 0
connectionTimeZoneIds: [GMT, Etc/GMT-0, Atlantic/St_Helena, Etc/GMT+0, Africa/Banjul, Etc/GMT, Africa/Freetown, Africa/Bamako, Africa/Conakry, Universal, Africa/Sao_Tome, Africa/Nouakchott, UTC, Etc/Universal, Atlantic/Azores, Africa/Abidjan, Africa/Accra, Etc/UCT, GMT0, Zulu, Africa/Ouagadougou, Atlantic/Reykjavik, Etc/Zulu, Iceland, Africa/Lome, Greenwich, Etc/GMT0, America/Danmarkshavn, Africa/Dakar, America/Scoresbysund, Africa/Bissau, Etc/Greenwich, Africa/Timbuktu, UCT, Africa/Monrovia, Etc/UTC]
connectionTimeZone: UTC
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2024-01-01 00:00:00.0
my_datetime   : 2024-01-01 00:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2024-01-01 03:00:00.0
my_datetime   : 2024-01-01 03:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2024-01-01 09:00:00.0
my_datetime   : 2024-01-01 09:00:00.0


client offset: 2
connectionTimeZoneIds: [Africa/Mbabane, Europe/Brussels, Europe/Warsaw, CET, Europe/Luxembourg, Etc/GMT-2, Libya, Africa/Kigali, Africa/Tripoli, Europe/Kaliningrad, Africa/Windhoek, Europe/Malta, Europe/Busingen, Europe/Skopje, Europe/Sarajevo, Europe/Rome, Europe/Zurich, Europe/Gibraltar, Africa/Lubumbashi, Europe/Vaduz, Europe/Ljubljana, Europe/Berlin, Europe/Stockholm, Europe/Budapest, Europe/Zagreb, Europe/Paris, Africa/Ceuta, Europe/Prague, Antarctica/Troll, Africa/Gaborone, Europe/Copenhagen, Europe/Vienna, Europe/Tirane, MET, Europe/Amsterdam, Africa/Maputo, Europe/San_Marino, Poland, Europe/Andorra, Europe/Oslo, Europe/Podgorica, Africa/Bujumbura, Atlantic/Jan_Mayen, Africa/Maseru, Europe/Madrid, Africa/Blantyre, Africa/Lusaka, Africa/Harare, Africa/Khartoum, Africa/Johannesburg, Europe/Belgrade, Africa/Juba, Europe/Bratislava, Arctic/Longyearbyen, Europe/Vatican, Europe/Monaco]
connectionTimeZone: Africa/Mbabane
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2023-12-31 22:00:00.0
my_datetime   : 2023-12-31 22:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2024-01-01 01:00:00.0
my_datetime   : 2024-01-01 01:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2024-01-01 07:00:00.0
my_datetime   : 2024-01-01 07:00:00.0


client offset: 3
connectionTimeZoneIds: [Asia/Aden, Africa/Nairobi, Africa/Cairo, Europe/Istanbul, Etc/GMT-3, Europe/Zaporozhye, Israel, Indian/Comoro, Antarctica/Syowa, Africa/Mogadishu, Europe/Bucharest, Africa/Asmera, Europe/Mariehamn, Asia/Istanbul, Europe/Tiraspol, Europe/Moscow, Europe/Chisinau, Europe/Helsinki, Asia/Beirut, Asia/Tel_Aviv, Africa/Djibouti, Europe/Simferopol, Europe/Sofia, Asia/Gaza, Africa/Asmara, Europe/Riga, Asia/Baghdad, Asia/Damascus, Africa/Dar_es_Salaam, Africa/Addis_Ababa, Europe/Uzhgorod, Asia/Jerusalem, Asia/Riyadh, Asia/Kuwait, Europe/Kirov, Africa/Kampala, Europe/Minsk, Asia/Qatar, Europe/Kiev, Asia/Bahrain, Europe/Vilnius, Indian/Antananarivo, Indian/Mayotte, Europe/Volgograd, Europe/Tallinn, Turkey, Europe/Kyiv, Asia/Nicosia, Asia/Famagusta, W-SU, EET, Asia/Hebron, Egypt, Asia/Amman, Europe/Nicosia, Europe/Athens]
connectionTimeZone: Asia/Aden
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2023-12-31 21:00:00.0
my_datetime   : 2023-12-31 21:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2024-01-01 00:00:00.0
my_datetime   : 2024-01-01 00:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2024-01-01 06:00:00.0
my_datetime   : 2024-01-01 06:00:00.0


client offset: 5
connectionTimeZoneIds: [Asia/Aqtau, Etc/GMT-5, Asia/Samarkand, Asia/Karachi, Asia/Yekaterinburg, Asia/Dushanbe, Indian/Maldives, Asia/Oral, Asia/Tashkent, Antarctica/Mawson, Asia/Qyzylorda, Asia/Aqtobe, Asia/Ashkhabad, Asia/Ashgabat, Asia/Atyrau, Indian/Kerguelen]
connectionTimeZone: Asia/Aqtau
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2023-12-31 19:00:00.0
my_datetime   : 2023-12-31 19:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2023-12-31 22:00:00.0
my_datetime   : 2023-12-31 22:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2024-01-01 04:00:00.0
my_datetime   : 2024-01-01 04:00:00.0


client offset: 9
connectionTimeZoneIds: [Etc/GMT-9, Pacific/Palau, Asia/Chita, Asia/Dili, Asia/Jayapura, Asia/Yakutsk, Asia/Pyongyang, ROK, Asia/Seoul, Asia/Khandyga, Japan, Asia/Tokyo]
connectionTimeZone: Asia/Seoul
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2023-12-31 15:00:00.0
my_datetime   : 2023-12-31 15:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2023-12-31 18:00:00.0
my_datetime   : 2023-12-31 18:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2024-01-01 00:00:00.0
my_datetime   : 2024-01-01 00:00:00.0


client offset: 11
connectionTimeZoneIds: [Pacific/Ponape, Pacific/Bougainville, Antarctica/Casey, Pacific/Pohnpei, Pacific/Efate, Pacific/Norfolk, Asia/Magadan, Pacific/Kosrae, Asia/Sakhalin, Pacific/Noumea, Etc/GMT-11, Asia/Srednekolymsk, Pacific/Guadalcanal]
connectionTimeZone: Pacific/Ponape
------------------------------------------------
oid: 1
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+0 UTC
my_timestamp  : 2023-12-31 13:00:00.0
my_datetime   : 2023-12-31 13:00:00.0
------------------------------------------------
oid: 2
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+3 Asia/Aden
my_timestamp  : 2023-12-31 16:00:00.0
my_datetime   : 2023-12-31 16:00:00.0
------------------------------------------------
oid: 3
server_timezone: UTC+3 Europe/Moscow
client_timezone: UTC+9 Asia/Seoul
my_timestamp  : 2023-12-31 22:00:00.0
my_datetime   : 2023-12-31 22:00:00.0
```





## DB에 저장되는 내부적인 Timestamp

- 로컬 타임존이 UTC+2 인 클라이언트가 JDBC URL에 `connectionTimeZone=LOCAL`을 지정하고 다음과 같은 쿼리로 타임존이 UTC+1 인 MySQL DB에 `2020-01-01 12:00:00`를 저장하면 어떻게 될까? Connector/J 5 와 Connector/J 8 을 비교해보자.

  ```java
  Statement st = conn.createStatement();
  st.executeUpdate("CREATE TABLE t1 (ts TIMESTAMP)");
  
  PreparedStatement ps = conn.prepareStatement("INSERT INTO t1 VALUES (?)");
  ps.setTimestamp(1, Timestamp.valueOf("2020-01-01 12:00:00"));
  ps.executeUpdate();
  ```

### Connector/J 5.1 드라이버

#### 저장

- Connector/J 5.1 드라이버는 클라이언트와의 연결에 사용된 타임존과 DB 서버 타임존의 차이인 `클라이언트 타임존(UTC+2) - DB서버 타임존(UTC+1)`인 UTC+1를 `2020-01-01 12:00`에 반영해서 내부적으로는 `2024-01-01 11:00:00Z`에 해당하는 타임스탬프 값이 TIMESTAMP 필드에 저장된다.
- `2020-01-01 12:00:00`에 DB 서버 타임존인 UTC+1 을 적용한 결과인 `2024-01-01 11:00:00Z`에 해당하는 타임스탬프 값이 TIMESTAMP 필드에 저장된다.

### 조회





### Connector/J 8.0 ~ 8.0.22 드라이버

#### 저장

- Connector/J 8.0.22 이하 드라이버는 클라이언트와의 연결에 사용된 타임존인 UTC+2를 `2020-01-01 12:00:00`에 반영해서 내부적으로는 `2020-01-01 10:00:00Z`에 해당하는 타임스탬프 값이 TIMESTAMP 필드에 저장된다.

#### 조회

- 이를 UTC+2인 클라이언트에서 조회하면, MySQL 8 에 저장된 `2020-01-01 10:00:00` 값에 UTC+2를 반영해서 `2020-01-01 12:00:00`가 반환되고,
- UTC+3인 다른 클라이언트에서 조회하면, MySQL 8 에 저장된 `2020-01-01 10:00:00` 값에 UTC+3를 반영해서 `2020-01-01 13:00:00`이 반환된다.




