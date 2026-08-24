# HEV Integration (HEV SOCKS5 Tunnel) — مستند خلاصه

این سند چکیده‌ای از نحوهٔ استفاده و یکپارچه‌سازی موتور native HEV (hev-socks5-tunnel) در AetherST را فراهم می‌کند: پیاده‌سازی JNI، تولید پیکربندی، و ارتباط با سرویس‌های Kotlin.

## اجزا و نقش‌ها
- Native HEV engine
  - قرار دارد در third_party/hev-socks5-tunnel (شامل کد C/C++).
  - وظیفه: پل‌زدن TUN ↔ SOCKS5، پردازش صفر-کپی بسته‌ها، و پشتیبانی از UDP/DNS بر بستر SOCKS5.
- JNI bridge
  - فایل: `app/src/main/cpp/hev_tun2socks_jni.c`
  - متد اصلی: `nativeStart(configStr, tunFd)` که `hev_socks5_tunnel_main_from_str` را فراخوانی می‌کند.
- Kotlin wrapper
  - `HevTun2SocksNative.kt` — بارگذاری کتابخانهٔ native و اعلان متدهای external.
  - `HevTun2SocksEngine.kt` — مدیریت چرخهٔ حیاتِ engine (start/stop/stats) در سطح Kotlin/Coroutine.
  - `HevTun2SocksConfig.kt` — تولید رشتهٔ پیکربندی YAML برای HEV (MTU، آدرس/پورت SOCKS، mapdns و تنظیمات زمان‌قطع).
- VPN service integration
  - `AetherVpnService.kt` — قبل از برقراری TUN، دسترسی به SOCKS5 محلی را بررسی می‌کند و سپس HEV یا bridge را انتخاب می‌کند.
  - `SocksTunBridge.kt` / `LocalSocksProxyServer.kt` — نقطه‌های دیگر مربوط به پل‌زدن و relay محلی.

## ساخت و درج native
- فایل اندروید: `app/src/main/cpp/Android.mk` شامل فراخوانی ماژول `third_party/hev-socks5-tunnel` و تعریف `hev-tun2socks-jni`.
- هنگام بیلد، کتابخانهٔ `libhev-tun2socks-jni.so` ساخته و توسط `System.loadLibrary("hev-tun2socks-jni")` بارگذاری می‌شود.

## پیکربندی HEV
- از `HevTun2SocksConfig.generate(...)` برای تولید پیکربندی متنی (YAML-like) استفاده می‌شود.
- مقادیر مهم: `mtu`, `socks address/port`, `mapdns` (به‌طور پیش‌فرض 198.18.0.2)، `log-level`, `connect-timeout`, `read-write-timeout`.

## نکات عملکرد و ایمنی
- HEV برای throughput بالا و مصرف پایین باتری طراحی شده است (zero-copy).
- AetherST قبل از استفاده از HEV، دسترسی و reachabilityِ SOCKS5 محلی را بررسی می‌کند تا از شکست‌های زودهنگام جلوگیری کند.
- هنگام استفاده از HEV در حالت TUN، دقت کنید که MTU، DNS و قوانین Routing به درستی تنظیم شوند تا از نشتی IP جلوگیری شود.

## مسیرهای مرجع در کد
- JNI: `app/src/main/cpp/hev_tun2socks_jni.c`
- Android.mk: `app/src/main/cpp/Android.mk`
- Kotlin wrappers:
  - `app/src/main/java/io/github/immaghzbad/aetherst/core/HevTun2SocksNative.kt`
  - `app/src/main/java/io/github/immaghzbad/aetherst/core/HevTun2SocksEngine.kt`
  - `app/src/main/java/io/github/immaghzbad/aetherst/core/HevTun2SocksConfig.kt`
- VPN service: `app/src/main/java/io/github/immaghzbad/aetherst/service/AetherVpnService.kt`

## پیشنهادات بهبود (اختیاری)
- افزودن مثالِ کاملِ پیکربندی برای چند معماری (arm64-v8a, armeabi-v7a, x86_64) در docs/
- نمونهٔ دستورات build برای بیلد native به‌صورت محلی (NDK) در `docs/BUILD-NATIVE.md`.
- تست‌های واحد/ادغام برای HevTun2SocksEngine (شبیه‌سازی فایل descriptor و رفتار start/stop).

---
این سند خلاصه است؛ اگر می‌خواهید بررسی عمیق‌تر (مثلاً استخراج تمام پیکربندی‌های runtime یا نمونه‌های لاگ) انجام دهم، بگویید تا آن‌را گسترش دهم.
