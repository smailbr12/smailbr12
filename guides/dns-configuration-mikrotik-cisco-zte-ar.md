# دليل إعداد DNS على أجهزة الشبكات

<div align="center">

![DNS & Networking](https://img.shields.io/badge/NETWORKING-DNS%20CONFIGURATION-06B6D4?style=for-the-badge&labelColor=0B1220)
![MikroTik](https://img.shields.io/badge/MIKROTIK-RouterOS-293239?style=for-the-badge&logo=mikrotik&logoColor=white)
![Cisco](https://img.shields.io/badge/CISCO-IOS%20%2F%20IOS%20XE-049FD9?style=for-the-badge&logo=cisco&logoColor=white)
![ZTE](https://img.shields.io/badge/ZTE-HOME%20GATEWAYS-0072CE?style=for-the-badge)

**إعداد DNS وفهم الفرق بين DNS الخاص بالراوتر وDNS الذي يتلقاه المستخدمون عبر DHCP**

</div>

> دليل عملي تعليمي. الأوامر أمثلة لشبكة مختبرية، ويجب استبدال العناوين بما يناسب تصميم شبكتك قبل التطبيق.

---

## الفهرس

1. [ما الذي نغيّره؟](#1-ما-الذي-نغيّره)
2. [خطة المختبر](#2-خطة-المختبر)
3. [MikroTik RouterOS](#3-mikrotik-routeros)
4. [Cisco IOS وIOS XE](#4-cisco-ios-وios-xe)
5. [ZTE وHuawei وD-Link](#5-zte--huawei--d-link)
6. [التحقق واستكشاف الأعطال](#6-التحقق-واستكشاف-الأعطال)
7. [أمان DNS](#7-أمان-dns)
8. [المراجع الرسمية](#8-المراجع-الرسمية)

## 1. ما الذي نغيّره؟

DNS يحوّل أسماء مثل `example.com` إلى عناوين IP. عند تغيير إعداد DNS على الراوتر، انتبه إلى وجود إعدادين مختلفين:

| الإعداد | وظيفته |
|:--|:--|
| **DNS Upstream / WAN** | خوادم DNS التي يسألها الراوتر نفسه أو التي يمرر إليها استعلامات العملاء. |
| **DNS الموزّع عبر DHCP / LAN** | عناوين DNS التي يرسلها الراوتر إلى أجهزة الشبكة عند حصولها على عنوان IP. |
| **DNS Cache على الراوتر** | يجعل الراوتر وسيطًا يستقبل استعلامات LAN ويعيد استخدامها من الذاكرة المؤقتة. |

**مثال:** يمكن أن يستخدم الراوتر خوادم عامة كـ upstream، بينما يحصل العملاء عبر DHCP على عنوان الراوتر `192.168.10.1`. في هذا التصميم يجب أن يسمح الراوتر باستعلامات DNS من شبكة LAN فقط.

## 2. خطة المختبر

نستخدم هذه القيم في الأمثلة:

| العنصر | قيمة المختبر |
|:--|:--|
| شبكة LAN | `192.168.10.0/24` |
| عنوان البوابة/الراوتر | `192.168.10.1` |
| DNS أساسي وثانوي | `1.1.1.1` و`1.0.0.1` |

العنوانان أعلاه مثال لخدمة Cloudflare Public DNS. يمكن استخدام DNS الخاص بمزود الخدمة أو خادم داخلي بدلًا منهما. [تفاصيل عناوين Cloudflare الرسمية](https://developers.cloudflare.com/1.1.1.1/ip-addresses/).

## 3. MikroTik RouterOS

الأوامر التالية مناسبة كمثال على RouterOS v7. قبل التغيير، صدّر نسخة احتياطية من الإعدادات، وتأكد من اسم شبكة DHCP وعنوانها في جهازك.

### الخيار A — الراوتر كـ DNS Cache لشبكة LAN

يضبط خوادم upstream، ويفعّل خدمة DNS للعملاء، ثم يعلن عنوان الراوتر عبر DHCP:

```routeros
/ip dns set servers=1.1.1.1,1.0.0.1 allow-remote-requests=yes
/ip dhcp-server network set [find where address="192.168.10.0/24"] dns-server=192.168.10.1 gateway=192.168.10.1
```

إذا لم يوجد إدخال DHCP للشبكة بعد، أنشئه بعد التأكد من أن الإعدادات الأخرى لديك تستخدم العناوين نفسها:

```routeros
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
```

### الخيار B — إرسال DNS العام مباشرةً إلى العملاء

استخدم هذا الخيار إذا كنت لا تريد أن يعمل MikroTik كـ DNS Cache للعملاء:

```routeros
/ip dhcp-server network set [find where address="192.168.10.0/24"] dns-server=1.1.1.1,1.0.0.1
```

في هذا الخيار لا تفعّل `allow-remote-requests` لأجل العملاء؛ فهم يتصلون بخوادم DNS مباشرةً.

### التحقق على MikroTik

```routeros
/ip dns print
/ip dhcp-server network print detail
/ip dns cache print
```

بعد تجديد عنوان DHCP على جهاز عميل، تحقق أن قائمة DNS لديه تعرض `192.168.10.1` في الخيار A، أو `1.1.1.1` و`1.0.0.1` في الخيار B.

## 4. Cisco IOS وIOS XE

أوامر `ip name-server` تضبط خوادم الأسماء التي يستخدمها جهاز Cisco لحل الأسماء. أما أمر `dns-server` داخل DHCP pool فيرسل عناوين DNS إلى العملاء.

### إعداد DNS للراوتر وإرساله لعملاء DHCP

مثال لجهاز Cisco يعمل DHCP لشبكة المختبر:

```text
enable
configure terminal
ip name-server 1.1.1.1 1.0.0.1
ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 1.1.1.1 1.0.0.1
end
copy running-config startup-config
```

> إذا كان DHCP يعمل على جهاز آخر، لا تنشئ DHCP pool ثانية. عدّل خيار DNS على خادم DHCP الفعلي بدلًا من ذلك. إذا كان لديك pool باسم آخر، استخدم اسمها بدل `LAN`.

### أوامر التحقق

```text
show running-config | include ip name-server
show running-config | section ip dhcp pool
show ip dhcp binding
ping example.com
```

إذا فشل `ping example.com` بينما نجح الاتصال بعنوان IP، افحص DNS. نجاح الأمر يعتمد أيضًا على سياسة ICMP والاتصال الخارجي.

## 5. ZTE / Huawei / D-Link

تختلف قوائم الإدارة وأسماء الحقول حسب **الموديل وإصدار البرنامج الثابت (Firmware) ومزود الخدمة**. أجهزة CPE المنزلية لا تملك أمر CLI موحدًا يمكن ضمانه لكل موديلات ZTE أو Huawei أو D-Link؛ لذلك لا تستخدم أوامر Telnet/SSH عشوائية من موديل آخر.

### خطوات عامة عبر واجهة الويب

1. اتصل بالراوتر وافتح عنوان بوابة الشبكة في المتصفح، مثل `192.168.1.1` أو `192.168.0.1` حسب إعداد جهازك.
2. سجّل الدخول ببيانات الإدارة الخاصة بك.
3. لتغيير DNS الذي يستخدمه الراوتر للاتصال بالإنترنت، ابحث عن **Internet / WAN / Broadband** ثم افتح اتصال الإنترنت النشط وابحث عن **DNS Mode** أو **Primary DNS / Secondary DNS**.
4. لتغيير DNS الذي يصل إلى أجهزة LAN، ابحث عن **Local Network / LAN / DHCP Server** ثم عن **DNS Server** أو **DHCP DNS**.
5. أدخل العناوين المطلوبة، احفظ التعديل، ثم جدّد اتصال DHCP على جهاز العميل.
6. تحقق من DNS الذي استلمه العميل قبل إجراء أي تعديل إضافي.

قد تظهر الخيارات في واجهة ZTE تحت **Network → LAN → DHCP Server** أو **Internet → WAN**، لكن هذه أسماء شائعة وليست مضمونة لكل إصدار. في أجهزة Huawei وD-Link اتبع الوظيفة نفسها حتى لو اختلف اسم القائمة.

**لإضافة خطوات دقيقة وصور مطابقة لجهازك:** يلزم رقم الموديل وإصدار Firmware، مثل `ZTE ZXHN H168N` أو الاسم المكتوب أسفل الجهاز. لا تشارك كلمة مرور الإدارة.

## 6. التحقق واستكشاف الأعطال

نفّذ الفحوصات بهذا الترتيب:

1. **افحص عنوان IP والبوابة وDNS على العميل.** في Windows:
   ```powershell
   ipconfig /all
   ```
2. **جدّد DHCP** بعد تغيير خيار DNS؛ الأجهزة القائمة قد تحتفظ بالإعداد القديم حتى تجديد الإيجار أو إعادة الاتصال.
3. **اختبر الوصول إلى عنوان IP** ثم اختبر اسم نطاق. إذا كان IP يعمل واسم النطاق لا يعمل، ركّز على DNS.
4. **افحص اتصال الراوتر بخادم DNS**، وتأكد من عدم وجود قواعد Firewall أو سياسات ISP تمنع UDP/TCP على المنفذ 53.
5. **امسح ذاكرة DNS على Windows** عند الحاجة:
   ```powershell
   ipconfig /flushdns
   ```
6. **في Cisco** راجع `show ip dhcp binding` وتأكد أن العميل أخذ العنوان المتوقع، ثم افحص DHCP pool.
7. **في MikroTik** راجع إعدادات DNS وDHCP network، ثم تحقق من قواعد Firewall التي تتحكم بطلبات DNS.

## 7. أمان DNS

- لا تترك DNS Cache مفتوحًا لطلبات الإنترنت من أي مصدر. عند تفعيل `allow-remote-requests=yes` في MikroTik، اسمح بطلبات DNS من LAN الموثوقة فقط واحجبها عن WAN.
- DNS التقليدي عبر المنفذ 53 لا يشفّر الاستعلامات تلقائيًا؛ اتبع سياسة الشبكة، ولا تغيّر DNS في شبكة مؤسسة دون موافقة المسؤول.
- استخدم عنوانًا أساسيًا وثانويًا موثوقين ويمكن الوصول إليهما من الراوتر.
- سجّل الإعدادات السابقة قبل التغيير، وغيّر إعدادًا واحدًا في كل مرة حتى يسهل الرجوع إذا تعطل حل الأسماء.
- لا تطبق أوامر DHCP على شبكة حقيقية قبل التحقق من subnet والبوابة واسم pool أو DHCP network.

## 8. المراجع الرسمية

- [MikroTik RouterOS — DNS](https://help.mikrotik.com/docs/display/ROS/DNS)
- [MikroTik RouterOS — DHCP](https://help.mikrotik.com/docs/display/ROS/DHCP)
- [Cisco IOS XE 17 — DHCP Server Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_config-dhcp-server-xe.html)
- [Cisco IOS IP Addressing Services — Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-i3.html)
- [Cloudflare 1.1.1.1 — Resolver addresses](https://developers.cloudflare.com/1.1.1.1/ip-addresses/)
- [ZTE Support](https://www.zte.com.cn/global/support)
- [Huawei Support](https://consumer.huawei.com/en/support/)
- [D-Link Support](https://support.dlink.com/)

---

<div align="center">

**إعداد DNS يبدأ بفهم الفرق بين خادم DNS الذي يستخدمه الراوتر وخادم DNS الذي يستلمه العميل.**

</div>
