# Time-Zone Offsets and ZoneIds List

Java 17 기준 Time-Zone Offset 별 ZoneId 목록은 다음과 같이 출력할 수 있다.
아래 예제에서는 `hh:mm:ss` 형식으로 초 단위까지 출력했지만, Offset 값은 보통 `hh:mm` 형식으로 표시하며 초 단위 값을 버림하면 된다.

```kotlin
ZoneId.getAvailableZoneIds()
    .groupBy { zoneId ->
        ZonedDateTime.now(ZoneId.of(zoneId)).offset.totalSeconds
    }
    .toSortedMap()
    .forEach { (totalSeconds, zoneIds) ->
        val duration = Duration.ofSeconds(totalSeconds.toLong());
        val hours = String.format("%+03d", duration.toHoursPart())
        val minutes = String.format("%02d", duration.toMinutesPart().absoluteValue)
        val seconds = String.format("%02d", duration.toSecondsPart().absoluteValue)

        println("UTC$hours:$minutes:$seconds $zoneIds")
    }
```

참고로 [ZoneOffset](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/ZoneOffset.html)의 validation에 걸리지 않는 valid range는 `[-18:00, +18:00]`이지만,  
2008년에 확장된 것 기준 실제로 Offset 데이터가 존재하는 actual range는 `[-12:00, +14:00]`이다.

결과

```
UTC-12:00:00 [Etc/GMT+12]
UTC-11:00:00 [Pacific/Pago_Pago, Pacific/Samoa, Pacific/Niue, US/Samoa, Etc/GMT+11, Pacific/Midway]
UTC-10:00:00 [Pacific/Honolulu, Pacific/Rarotonga, Pacific/Tahiti, Pacific/Johnston, US/Hawaii, SystemV/HST10, Etc/GMT+10]
UTC-09:30:00 [Pacific/Marquesas]
UTC-09:00:00 [Etc/GMT+9, Pacific/Gambier, America/Atka, SystemV/YST9, America/Adak, US/Aleutian]
UTC-08:00:00 [Etc/GMT+8, US/Alaska, America/Juneau, America/Metlakatla, America/Yakutat, Pacific/Pitcairn, America/Sitka, America/Anchorage, SystemV/PST8, America/Nome, SystemV/YST9YDT]
UTC-07:00:00 [Canada/Yukon, Etc/GMT+7, US/Arizona, Mexico/BajaSur, America/Dawson_Creek, Canada/Pacific, PST8PDT, America/Mazatlan, SystemV/MST7, America/Dawson, Mexico/BajaNorte, America/Tijuana, America/Creston, America/Hermosillo, America/Santa_Isabel, America/Vancouver, America/Ensenada, America/Phoenix, America/Whitehorse, America/Fort_Nelson, SystemV/PST8PDT, America/Los_Angeles, US/Pacific]
UTC-06:00:00 [America/El_Salvador, America/Guatemala, America/Belize, America/Managua, America/Tegucigalpa, Etc/GMT+6, Pacific/Easter, America/Regina, America/Denver, Mexico/General, Pacific/Galapagos, America/Yellowknife, America/Swift_Current, America/Inuvik, America/Ciudad_Juarez, America/Boise, America/Costa_Rica, MST7MDT, America/Monterrey, SystemV/CST6, America/Chihuahua, Chile/EasterIsland, US/Mountain, America/Mexico_City, America/Edmonton, America/Bahia_Banderas, Canada/Mountain, America/Cambridge_Bay, Navajo, SystemV/MST7MDT, America/Merida, Canada/Saskatchewan, America/Shiprock]
UTC-05:00:00 [America/Panama, America/Chicago, America/Eirunepe, Etc/GMT+5, America/Porto_Acre, America/Guayaquil, America/Rankin_Inlet, US/Central, America/Rainy_River, America/Indiana/Knox, America/North_Dakota/Beulah, America/Jamaica, America/Atikokan, America/Coral_Harbour, America/North_Dakota/Center, America/Cayman, America/Indiana/Tell_City, America/Ojinaga, America/Matamoros, CST6CDT, America/Knox_IN, America/Bogota, America/Menominee, America/Resolute, SystemV/EST5, Canada/Central, Brazil/Acre, America/Cancun, America/Lima, US/Indiana-Starke, America/Rio_Branco, SystemV/CST6CDT, Jamaica, America/North_Dakota/New_Salem, America/Winnipeg]
UTC-04:00:00 [America/Cuiaba, America/Marigot, America/Indiana/Petersburg, Chile/Continental, America/Grand_Turk, Cuba, Etc/GMT+4, America/Manaus, America/Fort_Wayne, America/St_Thomas, America/Anguilla, America/Havana, US/Michigan, America/Barbados, America/Louisville, America/Curacao, America/Guyana, America/Martinique, America/Puerto_Rico, America/Port_of_Spain, SystemV/AST4, America/Indiana/Vevay, America/Indiana/Vincennes, America/Kralendijk, America/Antigua, America/Indianapolis, America/Iqaluit, America/St_Vincent, America/Kentucky/Louisville, America/Dominica, America/Asuncion, EST5EDT, America/Nassau, America/Kentucky/Monticello, Brazil/West, America/Aruba, America/Indiana/Indianapolis, America/Santiago, America/La_Paz, America/Thunder_Bay, America/Indiana/Marengo, America/Blanc-Sablon, America/Santo_Domingo, US/Eastern, Canada/Eastern, America/Port-au-Prince, America/St_Barthelemy, America/Nipigon, US/East-Indiana, America/St_Lucia, America/Montserrat, America/Lower_Princes, America/Detroit, America/Tortola, America/Porto_Velho, America/Campo_Grande, America/Virgin, America/Pangnirtung, America/Montreal, America/Indiana/Winamac, America/Boa_Vista, America/Grenada, America/New_York, America/St_Kitts, America/Caracas, America/Guadeloupe, America/Toronto, SystemV/EST5EDT]
UTC-03:00:00 [America/Argentina/Catamarca, Canada/Atlantic, America/Argentina/Cordoba, America/Araguaina, America/Argentina/Salta, Etc/GMT+3, America/Montevideo, Brazil/East, America/Argentina/Mendoza, America/Argentina/Rio_Gallegos, America/Catamarca, America/Cordoba, America/Sao_Paulo, America/Argentina/Jujuy, America/Cayenne, America/Recife, America/Buenos_Aires, America/Paramaribo, America/Moncton, America/Mendoza, America/Santarem, Atlantic/Bermuda, America/Maceio, Atlantic/Stanley, America/Halifax, Antarctica/Rothera, America/Argentina/San_Luis, America/Argentina/Ushuaia, Antarctica/Palmer, America/Punta_Arenas, America/Glace_Bay, America/Fortaleza, America/Thule, America/Argentina/La_Rioja, America/Belem, America/Jujuy, America/Bahia, America/Goose_Bay, America/Argentina/San_Juan, America/Argentina/ComodRivadavia, America/Argentina/Tucuman, America/Rosario, SystemV/AST4ADT, America/Argentina/Buenos_Aires]
UTC-02:30:00 [America/St_Johns, Canada/Newfoundland]
UTC-02:00:00 [America/Miquelon, Etc/GMT+2, America/Noronha, Brazil/DeNoronha, Atlantic/South_Georgia]
UTC-01:00:00 [Etc/GMT+1, America/Godthab, Atlantic/Cape_Verde, America/Nuuk]
UTC+00:00:00 [GMT, Etc/GMT-0, Atlantic/St_Helena, Etc/GMT+0, Africa/Banjul, Etc/GMT, Africa/Freetown, Africa/Bamako, Africa/Conakry, Universal, Africa/Sao_Tome, Africa/Nouakchott, UTC, Etc/Universal, Atlantic/Azores, Africa/Abidjan, Africa/Accra, Etc/UCT, GMT0, Zulu, Africa/Ouagadougou, Atlantic/Reykjavik, Etc/Zulu, Iceland, Africa/Lome, Greenwich, Etc/GMT0, America/Danmarkshavn, Africa/Dakar, America/Scoresbysund, Africa/Bissau, Etc/Greenwich, Africa/Timbuktu, UCT, Africa/Monrovia, Etc/UTC]
UTC+01:00:00 [Europe/London, Etc/GMT-1, Europe/Jersey, Europe/Guernsey, Europe/Isle_of_Man, Africa/Tunis, Africa/Malabo, GB-Eire, Africa/Lagos, Africa/Algiers, GB, Portugal, Africa/Ndjamena, Atlantic/Faeroe, Eire, Atlantic/Faroe, Europe/Dublin, Africa/Libreville, Africa/El_Aaiun, Africa/Douala, Africa/Brazzaville, Africa/Porto-Novo, Atlantic/Madeira, Europe/Lisbon, Atlantic/Canary, Africa/Casablanca, Europe/Belfast, Africa/Luanda, Africa/Kinshasa, Africa/Bangui, WET, Africa/Niamey]
UTC+02:00:00 [Africa/Mbabane, Europe/Brussels, Europe/Warsaw, CET, Europe/Luxembourg, Etc/GMT-2, Libya, Africa/Kigali, Africa/Tripoli, Europe/Kaliningrad, Africa/Windhoek, Europe/Malta, Europe/Busingen, Europe/Skopje, Europe/Sarajevo, Europe/Rome, Europe/Zurich, Europe/Gibraltar, Africa/Lubumbashi, Europe/Vaduz, Europe/Ljubljana, Europe/Berlin, Europe/Stockholm, Europe/Budapest, Europe/Zagreb, Europe/Paris, Africa/Ceuta, Europe/Prague, Antarctica/Troll, Africa/Gaborone, Europe/Copenhagen, Europe/Vienna, Europe/Tirane, MET, Europe/Amsterdam, Africa/Maputo, Europe/San_Marino, Poland, Europe/Andorra, Europe/Oslo, Europe/Podgorica, Africa/Bujumbura, Atlantic/Jan_Mayen, Africa/Maseru, Europe/Madrid, Africa/Blantyre, Africa/Lusaka, Africa/Harare, Africa/Khartoum, Africa/Johannesburg, Europe/Belgrade, Africa/Juba, Europe/Bratislava, Arctic/Longyearbyen, Europe/Vatican, Europe/Monaco]
UTC+03:00:00 [Asia/Aden, Africa/Nairobi, Africa/Cairo, Europe/Istanbul, Etc/GMT-3, Europe/Zaporozhye, Israel, Indian/Comoro, Antarctica/Syowa, Africa/Mogadishu, Europe/Bucharest, Africa/Asmera, Europe/Mariehamn, Asia/Istanbul, Europe/Tiraspol, Europe/Moscow, Europe/Chisinau, Europe/Helsinki, Asia/Beirut, Asia/Tel_Aviv, Africa/Djibouti, Europe/Simferopol, Europe/Sofia, Asia/Gaza, Africa/Asmara, Europe/Riga, Asia/Baghdad, Asia/Damascus, Africa/Dar_es_Salaam, Africa/Addis_Ababa, Europe/Uzhgorod, Asia/Jerusalem, Asia/Riyadh, Asia/Kuwait, Europe/Kirov, Africa/Kampala, Europe/Minsk, Asia/Qatar, Europe/Kiev, Asia/Bahrain, Europe/Vilnius, Indian/Antananarivo, Indian/Mayotte, Europe/Volgograd, Europe/Tallinn, Turkey, Europe/Kyiv, Asia/Nicosia, Asia/Famagusta, W-SU, EET, Asia/Hebron, Egypt, Asia/Amman, Europe/Nicosia, Europe/Athens]
UTC+03:30:00 [Iran, Asia/Tehran]
UTC+04:00:00 [Asia/Yerevan, Etc/GMT-4, Asia/Dubai, Indian/Reunion, Indian/Mauritius, Europe/Saratov, Europe/Samara, Indian/Mahe, Asia/Baku, Asia/Muscat, Europe/Astrakhan, Asia/Tbilisi, Europe/Ulyanovsk]
UTC+04:30:00 [Asia/Kabul]
UTC+05:00:00 [Asia/Aqtau, Etc/GMT-5, Asia/Samarkand, Asia/Karachi, Asia/Yekaterinburg, Asia/Dushanbe, Indian/Maldives, Asia/Oral, Asia/Tashkent, Antarctica/Mawson, Asia/Qyzylorda, Asia/Aqtobe, Asia/Ashkhabad, Asia/Ashgabat, Asia/Atyrau, Indian/Kerguelen]
UTC+05:30:00 [Asia/Kolkata, Asia/Colombo, Asia/Calcutta]
UTC+05:45:00 [Asia/Kathmandu, Asia/Katmandu]
UTC+06:00:00 [Asia/Kashgar, Etc/GMT-6, Asia/Almaty, Asia/Dacca, Asia/Omsk, Asia/Dhaka, Indian/Chagos, Asia/Qostanay, Asia/Bishkek, Antarctica/Vostok, Asia/Urumqi, Asia/Thimbu, Asia/Thimphu]
UTC+06:30:00 [Asia/Yangon, Asia/Rangoon, Indian/Cocos]
UTC+07:00:00 [Asia/Pontianak, Etc/GMT-7, Asia/Phnom_Penh, Asia/Novosibirsk, Antarctica/Davis, Asia/Tomsk, Asia/Jakarta, Asia/Barnaul, Indian/Christmas, Asia/Ho_Chi_Minh, Asia/Hovd, Asia/Bangkok, Asia/Vientiane, Asia/Novokuznetsk, Asia/Krasnoyarsk, Asia/Saigon]
UTC+08:00:00 [Asia/Kuching, Asia/Chungking, Etc/GMT-8, Australia/Perth, Asia/Macao, Asia/Macau, Asia/Choibalsan, Asia/Shanghai, Asia/Ulan_Bator, Asia/Chongqing, Asia/Ulaanbaatar, Asia/Taipei, Asia/Manila, PRC, Asia/Ujung_Pandang, Asia/Harbin, Singapore, Asia/Brunei, Australia/West, Asia/Hong_Kong, Asia/Makassar, Hongkong, Asia/Kuala_Lumpur, Asia/Irkutsk, Asia/Singapore]
UTC+08:45:00 [Australia/Eucla]
UTC+09:00:00 [Etc/GMT-9, Pacific/Palau, Asia/Chita, Asia/Dili, Asia/Jayapura, Asia/Yakutsk, Asia/Pyongyang, ROK, Asia/Seoul, Asia/Khandyga, Japan, Asia/Tokyo]
UTC+09:30:00 [Australia/North, Australia/Yancowinna, Australia/Adelaide, Australia/Broken_Hill, Australia/South, Australia/Darwin]
UTC+10:00:00 [Australia/Hobart, Pacific/Yap, Australia/Tasmania, Pacific/Port_Moresby, Australia/ACT, Australia/Victoria, Antarctica/Macquarie, Pacific/Chuuk, Australia/Queensland, Australia/Canberra, Australia/Currie, Pacific/Guam, Pacific/Truk, Australia/NSW, Asia/Vladivostok, Pacific/Saipan, Antarctica/DumontDUrville, Australia/Sydney, Australia/Brisbane, Etc/GMT-10, Asia/Ust-Nera, Australia/Melbourne, Australia/Lindeman]
UTC+10:30:00 [Australia/Lord_Howe, Australia/LHI]
UTC+11:00:00 [Pacific/Ponape, Pacific/Bougainville, Antarctica/Casey, Pacific/Pohnpei, Pacific/Efate, Pacific/Norfolk, Asia/Magadan, Pacific/Kosrae, Asia/Sakhalin, Pacific/Noumea, Etc/GMT-11, Asia/Srednekolymsk, Pacific/Guadalcanal]
UTC+12:00:00 [Pacific/Kwajalein, Antarctica/McMurdo, Pacific/Wallis, Pacific/Fiji, Pacific/Funafuti, Pacific/Nauru, Kwajalein, NZ, Pacific/Wake, Antarctica/South_Pole, Pacific/Tarawa, Pacific/Auckland, Asia/Kamchatka, Etc/GMT-12, Asia/Anadyr, Pacific/Majuro]
UTC+12:45:00 [NZ-CHAT, Pacific/Chatham]
UTC+13:00:00 [Pacific/Fakaofo, Pacific/Enderbury, Pacific/Apia, Pacific/Kanton, Pacific/Tongatapu, Etc/GMT-13]
UTC+14:00:00 [Pacific/Kiritimati, Etc/GMT-14]
```

