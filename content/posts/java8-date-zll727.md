---
title: Java8 Date
slug: java8-date-zll727
url: /post/java8-date-zll727.html
date: '2024-04-20 08:47:40'
lastmod: '2024-04-20 15:19:43'
toc: true
isCJKLanguage: true
---



https://www.baeldung.com/java-8-date-time-intro

## 概述

Java8 之前使用 `java.util.Date`​ 和 `java.util.Calender`​ 来处理时间。

Java8 引入了新的日期和时间 API，由 `java.time`​ 包下的 LocalDate、LocalTime、LocalDateTime、ZonedDateTime、Period、Duration 等类支持。解决了之前 Date/Time API 的一些问题：

* Date 和 Calendar 不是线程安全的
* Date 和 Calendar API 无法支持常见时间操作
* 时区逻辑处理繁琐

关于 Date 和 Calendar 使用就不展开了，直接用 Java8 引入的新版时间 API 即可。

## LocalDate, LocalTime 和 LocalDateTime

当不需要显式指定时区时使用这些类。

### LocalDate

LocalDate 表示 ISO 格式（yyyy-MM-dd）的日期，不包括时间：

```java
System.out.println(LocalDate.now()); // 2024-04-20
```

‍

获取当前日期指定格式的字符串：

```java
String yyyyMMdd = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
System.out.println(yyyyMMdd); // 20240420
```

‍

使用 of 或 parse 获取指定年、月、日的 LocalDate：

```java
System.out.println(LocalDate.of(2024, 3, 31)); // 2024-03-31
```

‍

明天：

```java
System.out.println(LocalDate.now().plusDays(1)); // 2024-04-21
```

‍

上个月的同一天：

```java
System.out.println(LocalDate.of(2024, 5, 31).minus(1, ChronoUnit.MONTHS)); // 2024-04-30
System.out.println(LocalDate.of(2024, 5, 30).minusMonths(1)); // 2024-04-30
System.out.println(LocalDate.of(2024, 3, 31).minusMonths(1)); // 2024-02-29
```

如果当月比上月长，当月多出来的都会对应当上月的最后一天。

‍

星期几：

```java
DayOfWeek dayOfWeek = LocalDate.now().getDayOfWeek();
System.out.println(dayOfWeek); // SATURDAY
```

‍

这个月的第几天：

```java
int dayOfMonth = LocalDate.now().getDayOfMonth();
System.out.println(dayOfMonth); // 20
```

‍

是否是闰年：

```java
boolean leapYear = LocalDate.now().isLeapYear();
System.out.println(leapYear); // true
```

‍

日期的前后关系：

```java
boolean after = LocalDate.parse("2024-04-21").isAfter(LocalDate.parse("2024-04-20"));
System.out.println(after); // true
boolean before = LocalDate.parse("2024-04-21").isBefore(LocalDate.parse("2024-04-20"));
System.out.println(before); // false
```

‍

一天的开始时间：

```java
LocalDateTime atStartOfDay = LocalDate.now().atStartOfDay();
System.out.println(atStartOfDay); // 2024-04-20T00:00
```

‍

当前所在月的第一天：

```java
LocalDate firstDayOfMonth = LocalDate.now().with(TemporalAdjusters.firstDayOfMonth());
System.out.println(firstDayOfMonth); // 2024-04-01
```

### LocalTime

LocalTime 表示不带日期的时间，其 API 与 LocalDate 类似：

```java
LocalTime now = LocalTime.now();
System.out.println(now); // 15:51:15.969039
```

‍

获取当前时间指定格式的字符串：

```java
String nowStr = LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss"));
System.out.println(nowStr); // 16:00:44
```

‍

使用 of 或 parse 获取指定时、分、秒的 LocalTime：

```java
System.out.println(LocalTime.parse("18:00")); // 18:00
System.out.println(LocalTime.of(22, 45)); // 22:45
```

‍

上一个小时的相同时间：

```java
System.out.println(LocalTime.of(7, 30).minusHours(1)); // 06:30
```

‍

时间的前后关系：

```java
System.out.println(LocalTime.of(7, 30).isBefore(LocalTime.of(8, 0))); // true
```

‍

一天的开始和结束：

```java
System.out.println(LocalTime.MIN); // 00:00
System.out.println(LocalTime.MAX); // 23:59:59.999999999
```

‍

### LocalDateTime

LocalDateTime 表示日期和时间的组合：

```java
LocalDateTime now = LocalDateTime.now();
System.out.println(now); // 2024-04-20T16:13:27.481110
```

‍

获取指定格式的字符串：

```java
System.out.println(LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))); // 2024-04-20 16:15:10
```

‍

使用 of 或 parse 获取指定的 LocalDateTime：

```java
System.out.println(LocalDateTime.of(LocalDate.of(2023, 6, 30), LocalTime.of(22, 45))); // 2023-06-30T22:45
System.out.println(LocalDateTime.of(2023, 6, 30, 12, 34, 56)); // 2023-06-30T12:34:56
System.out.println(LocalDateTime.parse("2023-06-30T12:34:56")); // 2023-06-30T12:34:56
```

‍

LocalDateTime 上的运算操作：

```java
System.out.println(LocalDateTime.now().plusDays(1)); // 2024-04-21T16:32:43.381378
System.out.println(LocalDateTime.now().minusMonths(1)); // 2024-03-20T16:32:43.381409
```

‍

提取特定单位的值：

```java
 System.out.println(LocalDateTime.now().getYear()); // 2024
```

‍

## ZoneDateTime

当要处理特定时区的 Date 和 Time 时可以使用 ZoneDateTime。

列出所有的 zone Ids

```java
Set<String> availableZoneIds = ZoneId.getAvailableZoneIds();
System.out.println(availableZoneIds); // [Asia/Aden, America/Cuiaba, Etc/GMT+9, Etc/GMT+8, Africa/Nairobi, America/Marigot, Asia/Aqtau, Pacific/Kwajalein, America/El_Salvador, Asia/Pontianak, Africa/Cairo, Pacific/Pago_Pago, Africa/Mbabane, Asia/Kuching, Pacific/Honolulu, Pacific/Rarotonga, America/Guatemala, Australia/Hobart, Europe/London, America/Belize, America/Panama, Asia/Chungking, America/Managua, America/Indiana/Petersburg, Asia/Yerevan, Europe/Brussels, GMT, Europe/Warsaw, America/Chicago, Asia/Kashgar, Chile/Continental, Pacific/Yap, CET, Etc/GMT-1, Etc/GMT-0, Europe/Jersey, America/Tegucigalpa, Etc/GMT-5, Europe/Istanbul, America/Eirunepe, Etc/GMT-4, America/Miquelon, Etc/GMT-3, Europe/Luxembourg, Etc/GMT-2, Etc/GMT-9, America/Argentina/Catamarca, Etc/GMT-8, Etc/GMT-7, Etc/GMT-6, Europe/Zaporozhye, Canada/Yukon, Canada/Atlantic, Atlantic/St_Helena, Australia/Tasmania, Libya, Europe/Guernsey, America/Grand_Turk, Asia/Samarkand, America/Argentina/Cordoba, Asia/Phnom_Penh, Africa/Kigali, Asia/Almaty, US/Alaska, Asia/Dubai, Europe/Isle_of_Man, America/Araguaina, Cuba, Asia/Novosibirsk, America/Argentina/Salta, Etc/GMT+3, Africa/Tunis, Etc/GMT+2, Etc/GMT+1, Pacific/Fakaofo, Africa/Tripoli, Etc/GMT+0, Israel, Africa/Banjul, Etc/GMT+7, Indian/Comoro, Etc/GMT+6, Etc/GMT+5, Etc/GMT+4, Pacific/Port_Moresby, US/Arizona, Antarctica/Syowa, Indian/Reunion, Pacific/Palau, Europe/Kaliningrad, America/Montevideo, Africa/Windhoek, Asia/Karachi, Africa/Mogadishu, Australia/Perth, Brazil/East, Etc/GMT, Asia/Chita, Pacific/Easter, Antarctica/Davis, Antarctica/McMurdo, Asia/Macao, America/Manaus, Africa/Freetown, Europe/Bucharest, Asia/Tomsk, America/Argentina/Mendoza, Asia/Macau, Europe/Malta, Mexico/BajaSur, Pacific/Tahiti, Africa/Asmera, Europe/Busingen, America/Argentina/Rio_Gallegos, Africa/Malabo, Europe/Skopje, America/Catamarca, America/Godthab, Europe/Sarajevo, Australia/ACT, GB-Eire, Africa/Lagos, America/Cordoba, Europe/Rome, Asia/Dacca, Indian/Mauritius, Pacific/Samoa, America/Regina, America/Fort_Wayne, America/Dawson_Creek, Africa/Algiers, Europe/Mariehamn, America/St_Johns, America/St_Thomas, Europe/Zurich, America/Anguilla, Asia/Dili, America/Denver, Africa/Bamako, Europe/Saratov, GB, Mexico/General, Pacific/Wallis, Europe/Gibraltar, Africa/Conakry, Africa/Lubumbashi, Asia/Istanbul, America/Havana, NZ-CHAT, Asia/Choibalsan, America/Porto_Acre, Asia/Omsk, Europe/Vaduz, US/Michigan, Asia/Dhaka, America/Barbados, Europe/Tiraspol, Atlantic/Cape_Verde, Asia/Yekaterinburg, America/Louisville, Pacific/Johnston, Pacific/Chatham, Europe/Ljubljana, America/Sao_Paulo, Asia/Jayapura, America/Curacao, Asia/Dushanbe, America/Guyana, America/Guayaquil, America/Martinique, Portugal, Europe/Berlin, Europe/Moscow, Europe/Chisinau, America/Puerto_Rico, America/Rankin_Inlet, Pacific/Ponape, Europe/Stockholm, Europe/Budapest, America/Argentina/Jujuy, Australia/Eucla, Asia/Shanghai, Universal, Europe/Zagreb, America/Port_of_Spain, Europe/Helsinki, Asia/Beirut, Asia/Tel_Aviv, Pacific/Bougainville, US/Central, Africa/Sao_Tome, Indian/Chagos, America/Cayenne, Asia/Yakutsk, Pacific/Galapagos, Australia/North, Europe/Paris, Africa/Ndjamena, Pacific/Fiji, America/Rainy_River, Indian/Maldives, Australia/Yancowinna, SystemV/AST4, Asia/Oral, America/Yellowknife, Pacific/Enderbury, America/Juneau, Australia/Victoria, America/Indiana/Vevay, Asia/Tashkent, Asia/Jakarta, Africa/Ceuta, Asia/Barnaul, America/Recife, America/Buenos_Aires, America/Noronha, America/Swift_Current, Australia/Adelaide, America/Metlakatla, Africa/Djibouti, America/Paramaribo, Asia/Qostanay, Europe/Simferopol, Europe/Sofia, Africa/Nouakchott, Europe/Prague, America/Indiana/Vincennes, Antarctica/Mawson, America/Kralendijk, Antarctica/Troll, Europe/Samara, Indian/Christmas, America/Antigua, Pacific/Gambier, America/Indianapolis, America/Inuvik, America/Iqaluit, Pacific/Funafuti, UTC, Antarctica/Macquarie, Canada/Pacific, America/Moncton, Africa/Gaborone, Pacific/Chuuk, Asia/Pyongyang, America/St_Vincent, Asia/Gaza, Etc/Universal, PST8PDT, Atlantic/Faeroe, Asia/Qyzylorda, Canada/Newfoundland, America/Kentucky/Louisville, America/Yakutat, America/Ciudad_Juarez, Asia/Ho_Chi_Minh, Antarctica/Casey, Europe/Copenhagen, Africa/Asmara, Atlantic/Azores, Europe/Vienna, ROK, Pacific/Pitcairn, America/Mazatlan, Australia/Queensland, Pacific/Nauru, Europe/Tirane, Asia/Kolkata, SystemV/MST7, Australia/Canberra, MET, Australia/Broken_Hill, Europe/Riga, America/Dominica, Africa/Abidjan, America/Mendoza, America/Santarem, Kwajalein, America/Asuncion, Asia/Ulan_Bator, NZ, America/Boise, Australia/Currie, EST5EDT, Pacific/Guam, Pacific/Wake, Atlantic/Bermuda, America/Costa_Rica, America/Dawson, Asia/Chongqing, Eire, Europe/Amsterdam, America/Indiana/Knox, America/North_Dakota/Beulah, Africa/Accra, Atlantic/Faroe, Mexico/BajaNorte, America/Maceio, Etc/UCT, Pacific/Apia, GMT0, America/Atka, Pacific/Niue, Australia/Lord_Howe, Europe/Dublin, Pacific/Truk, MST7MDT, America/Monterrey, America/Nassau, America/Jamaica, Asia/Bishkek, America/Atikokan, Atlantic/Stanley, Australia/NSW, US/Hawaii, SystemV/CST6, Indian/Mahe, Asia/Aqtobe, America/Sitka, Asia/Vladivostok, Africa/Libreville, Africa/Maputo, Zulu, America/Kentucky/Monticello, Africa/El_Aaiun, Africa/Ouagadougou, America/Coral_Harbour, Pacific/Marquesas, Brazil/West, America/Aruba, America/North_Dakota/Center, America/Cayman, Asia/Ulaanbaatar, Asia/Baghdad, Europe/San_Marino, America/Indiana/Tell_City, America/Tijuana, Pacific/Saipan, SystemV/YST9, Africa/Douala, America/Chihuahua, America/Ojinaga, Asia/Hovd, America/Anchorage, Chile/EasterIsland, America/Halifax, Antarctica/Rothera, America/Indiana/Indianapolis, US/Mountain, Asia/Damascus, America/Argentina/San_Luis, America/Santiago, Asia/Baku, America/Argentina/Ushuaia, Atlantic/Reykjavik, Africa/Brazzaville, Africa/Porto-Novo, America/La_Paz, Antarctica/DumontDUrville, Asia/Taipei, Antarctica/South_Pole, Asia/Manila, Asia/Bangkok, Africa/Dar_es_Salaam, Poland, Atlantic/Madeira, Antarctica/Palmer, America/Thunder_Bay, Africa/Addis_Ababa, Asia/Yangon, Europe/Uzhgorod, Brazil/DeNoronha, Asia/Ashkhabad, Etc/Zulu, America/Indiana/Marengo, America/Creston, America/Punta_Arenas, America/Mexico_City, Antarctica/Vostok, Asia/Jerusalem, Europe/Andorra, US/Samoa, PRC, Asia/Vientiane, Pacific/Kiritimati, America/Matamoros, America/Blanc-Sablon, Asia/Riyadh, Iceland, Pacific/Pohnpei, Asia/Ujung_Pandang, Atlantic/South_Georgia, Europe/Lisbon, Asia/Harbin, Europe/Oslo, Asia/Novokuznetsk, CST6CDT, Atlantic/Canary, America/Knox_IN, Asia/Kuwait, SystemV/HST10, Pacific/Efate, Africa/Lome, America/Bogota, America/Menominee, America/Adak, Pacific/Norfolk, Europe/Kirov, America/Resolute, Pacific/Kanton, Pacific/Tarawa, Africa/Kampala, Asia/Krasnoyarsk, Greenwich, SystemV/EST5, America/Edmonton, Europe/Podgorica, Australia/South, Canada/Central, Africa/Bujumbura, America/Santo_Domingo, US/Eastern, Europe/Minsk, Pacific/Auckland, Africa/Casablanca, America/Glace_Bay, Canada/Eastern, Asia/Qatar, Europe/Kiev, Singapore, Asia/Magadan, SystemV/PST8, America/Port-au-Prince, Europe/Belfast, America/St_Barthelemy, Asia/Ashgabat, Africa/Luanda, America/Nipigon, Atlantic/Jan_Mayen, Brazil/Acre, Asia/Muscat, Asia/Bahrain, Europe/Vilnius, America/Fortaleza, Etc/GMT0, US/East-Indiana, America/Hermosillo, America/Cancun, Africa/Maseru, Pacific/Kosrae, Africa/Kinshasa, Asia/Kathmandu, Asia/Seoul, Australia/Sydney, America/Lima, Australia/LHI, America/St_Lucia, Europe/Madrid, America/Bahia_Banderas, America/Montserrat, Asia/Brunei, America/Santa_Isabel, Canada/Mountain, America/Cambridge_Bay, Asia/Colombo, Australia/West, Indian/Antananarivo, Australia/Brisbane, Indian/Mayotte, US/Indiana-Starke, Asia/Urumqi, US/Aleutian, Europe/Volgograd, America/Lower_Princes, America/Vancouver, Africa/Blantyre, America/Rio_Branco, America/Danmarkshavn, America/Detroit, America/Thule, Africa/Lusaka, Asia/Hong_Kong, Iran, America/Argentina/La_Rioja, Africa/Dakar, SystemV/CST6CDT, America/Tortola, America/Porto_Velho, Asia/Sakhalin, Etc/GMT+10, America/Scoresbysund, Asia/Kamchatka, Asia/Thimbu, Africa/Harare, Etc/GMT+12, Etc/GMT+11, Navajo, America/Nome, Europe/Tallinn, Turkey, Africa/Khartoum, Africa/Johannesburg, Africa/Bangui, Europe/Belgrade, Jamaica, Africa/Bissau, Asia/Tehran, WET, Europe/Astrakhan, Africa/Juba, America/Campo_Grande, America/Belem, Etc/Greenwich, Asia/Saigon, America/Ensenada, Pacific/Midway, America/Jujuy, Africa/Timbuktu, America/Bahia, America/Goose_Bay, America/Virgin, America/Pangnirtung, Asia/Katmandu, America/Phoenix, Africa/Niamey, America/Whitehorse, Pacific/Noumea, Asia/Tbilisi, Europe/Kyiv, America/Montreal, Asia/Makassar, America/Argentina/San_Juan, Hongkong, UCT, Asia/Nicosia, America/Indiana/Winamac, SystemV/MST7MDT, America/Argentina/ComodRivadavia, America/Boa_Vista, America/Grenada, Asia/Atyrau, Australia/Darwin, Asia/Khandyga, Asia/Kuala_Lumpur, Asia/Famagusta, Asia/Thimphu, Asia/Rangoon, Europe/Bratislava, Asia/Calcutta, America/Argentina/Tucuman, Asia/Kabul, Indian/Cocos, Japan, Pacific/Tongatapu, America/New_York, Etc/GMT-12, Etc/GMT-11, America/Nuuk, Etc/GMT-10, SystemV/YST9YDT, Europe/Ulyanovsk, Etc/GMT-14, Etc/GMT-13, W-SU, America/Merida, EET, America/Rosario, Canada/Saskatchewan, America/St_Kitts, Arctic/Longyearbyen, America/Fort_Nelson, America/Caracas, America/Guadeloupe, Asia/Hebron, Indian/Kerguelen, SystemV/PST8PDT, Africa/Monrovia, Asia/Ust-Nera, Egypt, Asia/Srednekolymsk, America/North_Dakota/New_Salem, Asia/Anadyr, Australia/Melbourne, Asia/Irkutsk, America/Shiprock, America/Winnipeg, Europe/Vatican, Asia/Amman, Etc/UTC, SystemV/AST4ADT, Asia/Tokyo, America/Toronto, Asia/Singapore, Australia/Lindeman, America/Los_Angeles, SystemV/EST5EDT, Pacific/Majuro, America/Argentina/Buenos_Aires, Europe/Nicosia, Pacific/Guadalcanal, Europe/Athens, US/Pacific, Europe/Monaco]
```

‍

转化 LocalDateTime 到特定时区：

```java
ZonedDateTime zonedDateTime = ZonedDateTime.of(LocalDateTime.now(), ZoneId.of("America/Los_Angeles"));
System.out.println(zonedDateTime); // 2024-04-20T16:41:05.766727-07:00[America/Los_Angeles]
```

‍

使用 of 或 parse 获取指定的 ZoneDateTime：

```java
ZonedDateTime zonedDateTime1 = ZonedDateTime.of(LocalDateTime.now(), ZoneId.of("America/Los_Angeles"));
System.out.println(zonedDateTime1); // 2024-04-20T16:43:33.696823-07:00[America/Los_Angeles]
ZonedDateTime parse = ZonedDateTime.parse("2023-04-20T16:41:05.766727-07:00[America/Los_Angeles]");
System.out.println(parse); // 2023-04-20T16:41:05.766727-07:00[America/Los_Angeles]
```

‍

表示特定时区的另一个方式是 OffsetDateTime：

```java
ZoneOffset zoneOffset = ZoneOffset.of("-07:00");
OffsetDateTime offsetDateTime = OffsetDateTime.of(LocalDateTime.now(), zoneOffset);
System.out.println(offsetDateTime); // 2024-04-20T16:46:10.556530-07:00
```

‍

## Period 和 Duration

Period 表示以年、月、日为单位的时间量。

Duration 表示以秒、纳秒为单位的时间量。

### Period

获取两个 LocalDate 之间的差：

```java
int days = Period.between(LocalDate.of(2023, 4, 1), LocalDate.now()).getDays();
System.out.println(days); // 19
System.out.println(Period.between(LocalDate.of(2023, 4, 1), LocalDate.now()).getYears()); // 1

long between = ChronoUnit.DAYS.between(LocalDate.of(2023, 4, 1), LocalDate.now());
System.out.println(between); // 385
```

Period.between().getDays() 表示两个 LocalDate 之间差了 几年、几月、几天 中的 几天。  
ChronoUnit.DAYS.between() 表示的就是我们直观认知的，累计起来的，两个 LocalDate 差了几天。

‍

### Duration

获取两个 LocalTime 之间的差：

```java
System.out.println(Duration.between(LocalTime.of(7, 30), LocalTime.now()).getSeconds()); // 34245
System.out.println(Duration.between(LocalTime.of(7, 30), LocalTime.now()).getNano()); // 975652000

System.out.println(ChronoUnit.SECONDS.between(LocalTime.of(7, 30), LocalTime.now())); // 34355
System.out.println(ChronoUnit.NANOS.between(LocalTime.of(7, 30), LocalTime.now())); // 34355975695000
```

这里的两种方法之间的差异和在 LocalDate 上类似，一个是只针对某个单位，一个是全部一起算。

‍

## 兼容 Date 和 Calendar

使用 toInstance 将 Date 和 Calendar 转为 Date 和 Time：

```java
Date date = new Date();
LocalDateTime localDateTime = LocalDateTime.ofInstant(date.toInstant(), ZoneId.systemDefault());
System.out.println(localDateTime); // 2024-04-20T17:09:21.023
    
LocalDate localDate = LocalDate.ofInstant(date.toInstant(), ZoneId.systemDefault());
System.out.println(localDate); // 2024-04-20
```

‍

使用 ofEpochSecond 将 Unix 时间戳转为 LocalDateTime：

```java
LocalDateTime localDateTime1 = LocalDateTime.ofEpochSecond(1713605025, 0, ZoneOffset.UTC);
System.out.println(localDateTime1); // 2024-04-20T09:23:45
```

‍
