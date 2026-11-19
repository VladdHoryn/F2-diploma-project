# ADR-001: Межа ізоляції для виконання недовіреного коду — OCI-контейнери через Docker Engine API; runc із SANDBOX-PROFILE як базовий runtime, gVisor (runsc) обов'язковий для публічного розгортання

| Атрибут | Значення |
|---|---|
| Статус | **Approved (Затверджено)** |
| Дата | 19.11.2026 |
| Автор / учасник рішення | Горин В. М. (автор кваліфікаційної роботи) |
| Рецензія | ШІ-суддя (ADR Reviewer), звіт J-01…J-25 — протокол AI Disclosure у PKM Workspace |
| Шаблон | M. Nygard (Title, Status, Context, Decision, Consequences) + Alternatives |
| Пов'язані артефакти | [`srs.md`](../srs.md) v1.3: [SRS-NFR-SEC-06](../srs.md#srs-nfr-sec-06)…08, SEC-19, [SRS-NFR-PERF-05](../srs.md#srs-nfr-perf-05), PERF-07, [SRS-NFR-REL-06](../srs.md#srs-nfr-rel-06), [SRS-DC-02](../srs.md#srs-dc-02), DC-05, DC-07, [табл. 2.3-2](../srs.md#table-2-3-2), ризик [R-01](../srs.md#r-01); [C4 L2 — контур виконання](../2_design.md#diagram-level2); [C4 L3 — порт `SandboxRuntime`, `DockerSandboxAdapter`, `SandboxConfigGuard`](../2_design.md#diagram-level3) |
| Заміщує / заміщено | — |
| Наступний перегляд | За тригерами розд. 7 |

---

## 1. Context

### 1.1 Проблема

Платформа приймає вихідний код кандидатів (Java 17, Python 3.12, C++17 — [SRS-DC-05](../srs.md#srs-dc-05)) і виконує його на інфраструктурі системи: компіляція, запуск на тест-кейсах, вимірювання часу і пам'яті. Код є недовіреним за визначенням (загроза Remote Code Execution — ключова інженерна проблема роботи). Потрібно обрати механізм, що задає межу ізоляції між кодом кандидата та хостом.

### 1.2 Модель загроз

| Ціль зловмисника (кандидата) | Приклад дії в коді | Наслідок без ізоляції |
|---|---|---|
| Вихід на хост | Експлуатація вразливості runtime або ядра | Доступ до PostgreSQL, RabbitMQ, Keycloak, секретів (усі на одному хості — [SRS-DC-02](../srs.md#srs-dc-02)) |
| Отримання прихованих тест-кейсів | Читання ФС, `/proc`, змінних оточення | Компрометація оцінювання |
| Відмова в обслуговуванні | Fork bomb, виділення пам'яті, нескінченний вивід | Деградація інших сесій і сервісів |
| Мережева активність | Сканування внутрішньої мережі, ексфільтрація | Атака на внутрішні сервіси |
| Персистентність | Залишені процеси або файли для наступного Job | Вплив на інших кандидатів |

### 1.3 Рушійні фактори (decision drivers) та обмеження SRS

| ID | Фактор | Джерело | Тип |
|---|---|---|---|
| D-1 | Сила межі ізоляції: що відділяє код від ядра хоста | [SRS-NFR-SEC-06](../srs.md#srs-nfr-sec-06)…08; R-01 | Пріоритетний |
| D-2 | Наскрізний час тривіального запуску p95 ≤ 3 с, включно зі створенням і видаленням sandbox | [SRS-NFR-PERF-05](../srs.md#srs-nfr-perf-05) | Жорстке обмеження |
| D-3 | Паралельність `worker.max-parallel` = 4 на REF-ENV (8 vCPU / 16 GiB, Ubuntu Server 24.04) | [SRS-NFR-PERF-07](../srs.md#srs-nfr-perf-07); розд. 3.1 | Жорстке обмеження |
| D-4 | Точність ліміту часу: запізнення ≤ 500 мс | [SRS-NFR-REL-06](../srs.md#srs-nfr-rel-06) | Жорстке обмеження |
| D-5 | Розгортання Docker Compose на одному хості без вимоги апаратної віртуалізації | [SRS-DC-02](../srs.md#srs-dc-02), DC-11 | Жорстке обмеження |
| D-6 | Жоден контейнер, крім docker-socket-proxy, не отримує Docker socket або `--privileged` | [SRS-NFR-SEC-06](../srs.md#srs-nfr-sec-06), SEC-07 | Жорстке обмеження |
| D-7 | Сумісність з контрактом Docker Engine API і портом `SandboxRuntime` | [SRS-IF-DOCKER-01](../srs.md#srs-if-docker-01); C4 L3 | Пріоритетний |
| D-8 | Вартість впровадження в межах графіка кваліфікаційної роботи (один розробник) | Обмеження проєкту | Пріоритетний |
| D-9 | Вимоги до хоста і вартість інфраструктури (FinOps) | Обмеження проєкту | Пріоритетний |

---

## 2. Alternatives

Рішення розділено на дві незалежні осі: **(а) інтерфейс керування sandbox** (Docker Engine API або власна оркестрація) і **(б) OCI-runtime, що визначає межу ізоляції**. runc, runsc (gVisor) і Kata підключаються до Docker Engine як альтернативні runtime і не потребують зміни інтерфейсу керування.

| Альтернатива | Межа ізоляції | Вимоги до хоста | Старт і накладні витрати | Інтеграція з D-7 |
|---|---|---|---|---|
| **A. Docker Engine + runc + SANDBOX-PROFILE** | Спільне ядро хоста; namespaces, cgroups v2, seccomp, cap-drop ALL, read-only rootfs, non-root | Будь-який Linux з Docker Engine | Базовий рівень; CPU — нативно | Повна (поточна реалізація) |
| **B. Docker Engine + gVisor (runsc)** | Ядро в просторі користувача (Sentry) перехоплює системні виклики; хостове ядро бачить обмежений набір викликів | Linux x86_64 / ARM64, ядро 4.14.77+; штатно не підтримується на Windows / macOS | CPU-bound — нативно; syscall- і ФС-інтенсивні навантаження — суттєві накладні витрати; ≈ 30 MiB пам'яті на sandbox | Повна: `--runtime=runsc`, реєстрація `runsc install` |
| **C. Docker / containerd + Kata Containers** | Апаратна віртуалізація: окреме гостьове ядро в легкій VM | KVM: bare metal або вкладена віртуалізація; не працює в Docker Desktop без вкладеної віртуалізації | Вищі, ніж у A/B: старт VM і пам'ять гостьового ядра на кожен sandbox | Часткова: runtime для Docker, але обмеження пристроїв і монтувань |
| **D. Firecracker microVM (напряму або firecracker-containerd)** | Апаратна віртуалізація; мінімальна модель пристроїв | KVM (`/dev/kvm`); гостьове ядро і rootfs на кожну мову | ≤ 125 мс до user space; < 5 MiB накладних витрат VMM | Відсутня: власний адаптер `SandboxRuntime`, власний конвеєр образів |
| **E. Процесна пісочниця (isolate / nsjail) у контейнері Worker** | Namespaces і cgroups, якими керує sandbox-утиліта | Керування cgroups зсередини контейнера: на практиці `--privileged` (Judge0) або `CAP_SYS_ADMIN` | Мінімальні | Відсутня: окремий механізм замість Docker API |

**Відсіювання за жорсткими обмеженнями:**

| Альтернатива | D-2 | D-3 | D-5 | D-6 | Результат |
|---|---|---|---|---|---|
| A | Так | Так | Так | Так | Допустима |
| B | Підлягає вимірюванню (компіляція — ФС-інтенсивна) | Так (+ ≈ 120 MiB на 4 sandbox) | Так (REF-ENV — Linux) | Так | Допустима за умови вимірювання D-2 |
| C | Підлягає вимірюванню | Знижує паралельність через пам'ять гостьових VM | **Ні**: потрібна апаратна віртуалізація | Так | Відхилена для v1.0 |
| D | Так | Так | **Ні**: потрібен KVM | Так | Відхилена для v1.0 |
| E | Так | Так | Так | **Ні**: привілейований контейнер Worker | Відхилена |

---

## 3. Decision

1. **Інтерфейс керування.** Sandbox є OCI-контейнером, яким Execution Worker керує через Docker Engine API (через docker-socket-proxy) за портом `SandboxRuntime`. Кожен Execution Job отримує окремий ефемерний контейнер.
2. **Базовий runtime — runc.** Для середовищ розробки та закритого REF-ENV використовується runc. Профіль SANDBOX-PROFILE ([табл. 2.3-2](../srs.md#table-2-3-2) SRS) обов'язковий для будь-якого runtime і перевіряється `SandboxConfigGuard` перед кожним створенням контейнера.
3. **gVisor (runsc) — обов'язковий для публічного розгортання.** Будь-яке розгортання, доступне користувачам поза командою розробки (проведення реальних співбесід), використовує `sandbox.runtime=runsc`. Execution Worker у профілі розгортання `public` не стартує з іншим runtime (fail-fast, [SRS-NFR-SEC-19](../srs.md#srs-nfr-sec-19)).
4. **Мінімальна версія runc.** Execution Worker при старті зчитує версії компонентів Docker Engine (`GET /version` через docker-socket-proxy) і не стартує, якщо версія runc нижча за `sandbox.runc-min-version`. Значення відповідає першій версії з виправленнями CVE-2025-31133, CVE-2025-52565, CVE-2025-52881 для встановленої гілки runc.
5. **Порівняльне вимірювання як частина роботи.** Метрики PERF-05, PERF-07 і набір SBX-ATTACK виконуються на REF-ENV для runc і runsc. Результати фіксуються в розділі експериментів кваліфікаційної роботи.
6. **Відхилено для v1.0:** C (Kata) і D (Firecracker) — вимога апаратної віртуалізації суперечить [SRS-DC-02](../srs.md#srs-dc-02); E (isolate / nsjail) — вимагає привілейованого контейнера, що суперечить [SRS-NFR-SEC-06](../srs.md#srs-nfr-sec-06), SEC-07. Шлях міграції на C/D збережено через порт `SandboxRuntime` (розд. 6).

**Обґрунтування.** Альтернатива A — єдина, що задовольняє всі жорсткі обмеження без вимірювань, і вже реалізована в моделі C4 L3. Її слабке місце — спільне ядро хоста (D-1) — підтверджене уразливостями runc 2024–2025 років. Тому A прийнято лише для середовищ без зовнішніх користувачів. Для публічного розгортання обрано B, що зменшує поверхню ядра хоста без вимоги апаратної віртуалізації і без зміни інтерфейсу керування (D-5, D-7). Вартість B — накладні витрати на системні виклики й операції ФС, тобто на компіляцію. Тому B не призначено за замовчуванням до вимірювання D-2 на REF-ENV (п. 5).

---

## 4. Consequences

### 4.1 Позитивні

| ID | Наслідок | Метрика / підтвердження |
|---|---|---|
| P-1 | Немає вимоги до апаратної віртуалізації: будь-який Linux-хост з Docker Engine | REF-ENV без KVM; стандартний VPS / віртуальний інстанс |
| P-2 | Один інтерфейс керування для runc і runsc; перемикання — параметр `sandbox.runtime` | 0 змін коду адаптера при зміні runtime |
| P-3 | Мінімальні накладні витрати базового runtime | PERF-05 (p95 ≤ 3 с) досяжний без пулу попередньо створених контейнерів |
| P-4 | Висока щільність: пам'ять sandbox визначається лімітом SANDBOX-PROFILE | `worker.max-parallel` = 4 на REF-ENV (формула SRS розд. 3.1) |
| P-5 | Оборотність: перехід на Kata / Firecracker обмежений новим адаптером порту `SandboxRuntime` | Ядро і прикладний шар Worker не змінюються (правило ArchUnit, [SRS-NFR-MNT-10](../srs.md#srs-nfr-mnt-10)) |
| P-6 | Порівняння runc / runsc на однакових сценаріях дає експериментальний результат для роботи | Звіт вимірювань PERF-05, PERF-07, SBX-ATTACK для двох runtime |

### 4.2 Негативні

| ID | Наслідок | Вплив | Компенсація |
|---|---|---|---|
| N-1 | **Спільне ядро хоста при runc.** Вразливість runc або ядра дає вихід із sandbox на хост, де працюють PostgreSQL, RabbitMQ, Keycloak ([SRS-DC-02](../srs.md#srs-dc-02)) | Повна компрометація системи | runc лише без зовнішніх користувачів; runsc — у `public`; SANDBOX-PROFILE; ризик [R-01](../srs.md#r-01) |
| N-2 | **Шлях `exec` після виконання недовіреного коду.** Тест-кейси запускаються процесами в контейнері, де вже виконувався код кандидата; це клас уразливостей CVE-2024-21626 | Вихід із sandbox при невиправленому runc | Перевірка мінімальної версії runc при старті (п. 4 Decision); довірений runner — у беклозі (J-24) |
| N-3 | **Обов'язок патч-менеджменту.** Валідність рішення залежить від своєчасного оновлення runc, containerd, ядра | Без оновлень ризик N-1 зростає з кожною новою CVE | SLA: оновлення runc / ядра з виправленням High / Critical CVE — до 7 днів; Trivy для образів ([SRS-NFR-SEC-09](../srs.md#srs-nfr-sec-09)) |
| N-4 | **Накладні витрати runsc на компіляцію.** `javac` / `g++` інтенсивно працюють із ФС; за незалежними вимірюваннями накладні витрати gVisor на syscall-інтенсивних навантаженнях сягають 56–125 % | Ризик невиконання PERF-05 у профілі `public` | Вимірювання на REF-ENV (п. 5 Decision); за невиконання — перегляд бюджету PERF-05 для `public` або окремий компілятор-контейнер |
| N-5 | **Сумісність runsc.** gVisor реалізує підмножину системних викликів; окремі інструменти мовних образів можуть працювати некоректно | Хибні RUNTIME_ERROR у `public` | Еталонні рішення (SRS розд. 3.4: 3 мови × 6 вердиктів) виконуються в CI на обох runtime |
| N-6 | **Різні середовища розробки та експлуатації.** На Windows / macOS (Docker Desktop) runsc штатно недоступний; розробка ведеться на runc | Дефекти, специфічні для runsc, виявляються пізніше | Окремий CI-етап на Linux-раннері з runsc (nightly) |
| N-7 | **Привілейований доступ Worker до Docker Engine.** Керування контейнерами вимагає API, еквівалентного root на хості | Компрометація Worker = ризик для хоста | docker-socket-proxy з allowlist ([SRS-NFR-SEC-07](../srs.md#srs-nfr-sec-07)); Worker без вхідного API для користувачів (C4 L3) |
| N-8 | **Vendor lock-in на Docker Engine API (Moby) і docker-java.** Перехід на containerd / CRI потребує нового адаптера | Вартість виходу — переписування `DockerSandboxAdapter`, `SandboxSpecFactory`, `SandboxConfigGuard` | Порт `SandboxRuntime`; типи docker-java не виходять за межі адаптера (ArchUnit) |
| N-9 | **CI/CD.** Інтеграційні тести sandbox потребують Docker Engine на CI-раннері; три мовні образи — ланцюжок постачання | Довший конвеєр; ризик уразливих образів | Кешування образів; Trivy ([SRS-NFR-SEC-09](../srs.md#srs-nfr-sec-09)); попереднє завантаження образів (припущення A-03) |

### 4.3 FinOps

| Аспект | A / B (обрано) | C / D (відхилено) |
|---|---|---|
| Тип хоста | Будь-який Linux-хост або віртуальний інстанс | Bare metal або інстанс із вкладеною віртуалізацією. В AWS вкладена віртуалізація на віртуальних інстансах доступна з 16.02.2026 для обмеженого набору сімейств; для чутливих до продуктивності навантажень AWS рекомендує bare metal |
| Додаткова пам'ять | B: ≈ 30 MiB на sandbox (≈ 120 MiB на 4 паралельні) | Гостьове ядро і пам'ять VM на кожен sandbox |
| Операційні витрати | Оновлення runc / runsc / ядра | Додатково: гостьові ядра, rootfs для кожної мови, VMM |

---

## 5. Компенсаційні заходи (обов'язкові умови рішення)

| Захід | Реалізація | Перевірка |
|---|---|---|
| SANDBOX-PROFILE для кожного контейнера | `SandboxSpecFactory`, `SandboxConfigGuard` (C4 L3) | [SRS-NFR-SEC-06](../srs.md#srs-nfr-sec-06), SBX-ATTACK |
| Мінімальна версія runc | Перевірка `GET /version` при старті Worker | [SRS-NFR-SEC-19](../srs.md#srs-nfr-sec-19) |
| runsc у профілі `public` | Fail-fast при `sandbox.runtime ≠ runsc` | [SRS-NFR-SEC-19](../srs.md#srs-nfr-sec-19) |
| Ізоляція Docker API | docker-socket-proxy з allowlist | [SRS-NFR-SEC-07](../srs.md#srs-nfr-sec-07) |
| Патч-менеджмент | SLA 7 днів для High / Critical CVE runc, containerd, ядра | Журнал оновлень хоста |

---

## 6. Шлях міграції

| Подія | Дія | Обсяг змін |
|---|---|---|
| runsc задовольняє PERF-05 на REF-ENV | runsc стає runtime за замовчуванням для всіх Linux-середовищ | Конфігурація `sandbox.runtime` |
| Багатовузлове розгортання з KVM-хостами | Новий адаптер `SandboxRuntime` для Kata або Firecracker | Пакет `adapter/out/…`; ядро без змін |
| Нова High / Critical CVE runsc | Тимчасовий перехід `public` у режим недоступності запусків до оновлення | Операційна процедура |

## 7. Тригери перегляду ADR

- результати вимірювань PERF-05 для runsc на REF-ENV (п. 5 Decision);
- публікація High / Critical CVE для runc або gVisor без виправлення протягом 7 днів;
- рішення про публічну експлуатацію поза кваліфікаційною роботою;
- перехід від одновузлового розгортання ([SRS-DC-02](../srs.md#srs-dc-02)) до кластера.

## 8. Джерела

- gVisor Performance Guide — https://gvisor.dev/docs/architecture_guide/performance/
- gVisor in Production at Scale in Ant — https://gvisor.dev/blog/2021/12/02/running-gvisor-in-production-at-scale-in-ant/
- gVisor Installation / Docker Quick Start — https://gvisor.dev/docs/user_guide/install, https://gvisor.dev/docs/user_guide/docker
- Огляд вимірювань gVisor (KubeBlocks) — https://jacar.es/?p=241
- gVisor vs Kata Containers vs runc — https://computingforgeeks.com/gvisor-vs-kata-containers-vs-runc/
- Firecracker — https://lwn.net/Articles/775736, https://northflank.com/blog/firecracker-vs-cloud-hypervisor
- Kata Containers: вимоги — https://katacontainers.io/software
- gVisor у Docker Desktop — https://dev.to/rimelek/using-gvisors-container-runtime-in-docker-desktop-374m
- CNCF: runc breakout vulnerabilities (2025) — https://www.cncf.io/blog/2025/11/28/runc-container-breakout-vulnerabilities-a-technical-overview/
- oss-sec: CVE-2025-31133, CVE-2025-52565, CVE-2025-52881 — https://seclists.org/oss-sec/2025/q4/138
- Red Hat RHSB-2024-001 (CVE-2024-21626) — https://access.redhat.com/security/vulnerability/RHSB-2024-001
- Judge0 sandbox escape — https://tantosec.com/blog/judge0/
- AWS: nested virtualization on virtual EC2 instances — https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-ec2-nested-virtualization-on-virtual
