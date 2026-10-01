# Miesięczny audyt techniczny bezpieczeństwa i RODO — Trippics, Yalquo, Mata24

**Data oceny:** 2026-10-01 (Europe/Warsaw, UTC+02:00) / 2026-10-01 UTC

**Tryb:** wyłącznie pasywny, punktowy przegląd techniczny

**Klasyfikacja raportu:** publiczny; dowody zredagowane

> **Zastrzeżenie:** raport jest techniczną oceną punktową opartą na dostępnych dowodach. Nie stanowi formalnej porady prawnej, audytu zgodności prawnej ani pełnego testu penetracyjnego. Nie wykonywano exploitów, prób logowania, fuzzingu, testów obciążeniowych ani kodu z audytowanych repozytoriów.

## 1. Executive summary

Audyt objął statycznie 12 niepustych repozytoriów, dwa puste repozytoria, odczyt dostępnych metadanych GitHub oraz bezpieczne, niesekretne pola stanu klastra Kubernetes. Nie uruchamiano kodu projektu, buildów, testów, kontenerów ani instalacji zależności.

Najpoważniejszym nowym problemem jest śledzony w bieżącej gałęzi backendu Trippics eksport relacyjnej bazy danych. Zredagowana analiza potwierdziła tabelę użytkowników z 110 rekordami, 110 unikalnymi adresami e-mail i 83 unikalnymi fingerprintami wartości o formacie starszych hashy haseł. Żadnej wartości nie użyto, nie zweryfikowano i nie zamieszczono w raporcie. Sytuacja wymaga obsługi jak incydent danych i poświadczeń, usunięcia także z historii Git oraz oceny obowiązków związanych z naruszeniem.

Od poprzedniego audytu potwierdzono kilka istotnych ulepszeń: Trippics przestał wysyłać hasło konta społecznościowego e-mailem; Mata24 wdrożył dwustronną redakcję raportów awarii i 30-dniową retencję; Yalquo dodał samoobsługową anonimizację konta; workloady aplikacyjne wyłączyły automatyczny token ServiceAccount; dodano deklaracje NetworkPolicy; jeden obraz Mata24 został przypięty do digestu. Większość workloadów nadal nie deklaruje pełnego `securityContext`, a pozostałe obrazy aplikacyjne i obrazy używane w CI są identyfikowane zmiennymi tagami.

W chwili odczytu osiem z dziewięciu objętych aplikacji Argo CD było `Synced` i `Healthy`; aplikacja backendowa Mata24 była `OutOfSync`, lecz `Healthy`, przy tej samej rewizji GitOps. Stan wymaga ponownego, pasywnego sprawdzenia po zakończeniu synchronizacji. API klastra pozwoliło odczytać workloady, ingressy i Argo CD, ale odmówiło odczytu NetworkPolicy i RBAC. API GitHub odmówiło dostępu do ustawień administracyjnych, alertów Dependabot, workflowów i rulesets. Kontrole te pozostają lukami pokrycia.

### Rozkład otwartych findingów

| Severity | Liczba |
|---|---:|
| Critical | 1 |
| High | 1 |
| Medium | 13 |
| Low | 3 |
| Info | 1 |
| **Razem** | **19** |

### Zweryfikowane zmiany od poprzedniego audytu

| ID | Wynik punktowej weryfikacji |
|---|---|
| MAT-SUP-001 | Naprawione w bieżącym drzewie `matematicon`; historyczne dumpy nie są już obecne. Pełna niezależna walidacja wszystkich refów i rotacji poświadczeń pozostaje poza dostępnym dowodem. |
| TRI-SEC-001 | Naprawione statycznie: losowe hasło konta Facebook nie jest wysyłane e-mailem. |
| MAT-RODO-001 | Naprawione statycznie: redakcja po stronie mobile i backendu, retencja 30 dni oraz aktualizacja informacji dla użytkownika. |
| YAL-RODO-002 | Częściowo: dodano samoobsługowe usunięcie/anonimizację konta, ale nadal nie znaleziono eksportu danych. |
| SHR-K8S-001 | Częściowo: token ServiceAccount wyłączono dla workloadów aplikacyjnych; pełny hardening kontenerów nadal nie jest powszechny. |
| SHR-K8S-002 | Częściowo: deklaracje NetworkPolicy są w GitOps; brak uprawnienia do potwierdzenia obiektów live. |
| SHR-SUP-001 | Częściowo: jeden workload przypięto do digestu; pozostałe używają tagów. |

## 2. Zakres i metodologia

### Repozytoria

- Trippics: `trippics`, `trippics-backend`, `trippics-frontend`, `trippics-mobile`.
- Yalquo: `yalquo-backend`, `yalquo-frontend`, `yalquo-mobile`, `yalquo-bot`.
- Mata24: `matematicon`, `mata24-frontend`, `mata24-mobile`, `mata24-marketing`, `mata24-bot`.
- Wspólna infrastruktura: `local-kubernetes-cluster-definition`.
- Raporty: `local-kubernetes-audits`.

`yalquo-mobile` i `mata24-marketing` są puste — nie mają commitów ani gałęzi. Nie można było wykonać kontroli kodu tych komponentów.

### Metoda

1. Zanotowano czas UTC i Europe/Warsaw oraz SHA domyślnych gałęzi.
2. Repozytoria pobrano jako płytkie repozytoria bare z pustym `hooksPath`; zawartość wyeksportowano przez `git archive`. Nie wykonywano checkout filters, hooków ani kodu projektu.
3. Statycznie przejrzano kod, dependency manifests i lockfile, Dockerfile, Jenkinsfile, manifesty GitOps, dokumenty prawne i konfiguracje aplikacji mobilnych.
4. Lokalny heurystyczny skan sekretów i PII zapisywał wyłącznie typ, lokalizację, liczbę oraz 12-znakowy prefiks SHA-256; żadnej wartości nie użyto ani nie opublikowano.
5. GitHub API wykorzystano tylko do odczytu. Ustawienia administracyjne i bezpieczeństwa zwracały `403`.
6. `kubectl` wykorzystano wyłącznie do `get` niesekretnych metadanych/spec/status. Nie odczytywano Secrets, wartości ConfigMap, logów, kubeconfigu ani tokenów ServiceAccount.
7. Deklaracje GitOps porównano z bezpiecznymi polami live: klasy referencji obrazów, sondy, zasoby, ServiceAccount, `securityContext`, stan Argo CD i ingress/TLS.
8. Nie używano zewnętrznych skanerów przesyłających kod. W środowisku nie było lokalnych baz SCA ani narzędzi SAST/SBOM.

Statusy: **Confirmed** — dowód bezpośredni; **Likely** — silny dowód statyczny bez testu runtime; **Needs evidence** — brak wystarczającego bezpiecznego dowodu.

## 3. Coverage matrix

Legenda: ✅ wykonane, ◐ częściowe, ❌ niewykonane / brak bezpiecznego dowodu.

| Obszar | Trippics | Yalquo | Mata24 | Shared | Uwagi |
|---|:---:|:---:|:---:|:---:|---|
| AuthN/AuthZ, JWT, hasła | ✅ | ✅ | ✅ | — | Statyczny przegląd filtrów, kontrolerów i encoderów; bez prób logowania. |
| IDOR / własność obiektów | ◐ | ◐ | ◐ | — | Przegląd kontrolerów i serwisów; brak dynamicznej weryfikacji. |
| CORS/CSRF/XSS/injection | ◐ | ◐ | ◐ | — | Analiza statyczna; brak DAST i fuzzingu. |
| SSRF/path traversal/upload | ◐ | ◐ | ◐ | — | Analiza przepływów i walidacji; bez wysyłania plików. |
| Rate limiting i błędy | ◐ | ◐ | ◐ | — | Nie znaleziono aplikacyjnego limitera; brak bezpiecznego dowodu konfiguracji globalnej. |
| Dependencies/lockfile | ◐ | ◐ | ◐ | ◐ | Manifesty i npm lockfile przejrzane; brak lokalnej bazy CVE/OSV i alertów Dependabot. |
| Sekrety/PII w repo | ✅ | ✅ | ✅ | ✅ | Lokalny skan heurystyczny; bez walidowania znalezionych wartości. |
| GitHub/CI/CD | ◐ | ◐ | ◐ | ◐ | Jenkinsfile przejrzane; API ustawień GitHub zwracało `403`. |
| Docker/Kubernetes/GitOps | ◐ | ◐ | ◐ | ◐ | Workloady, ingress i Argo CD odczytane; NetworkPolicy/RBAC niedostępne live. |
| Backup/restore | ◐ | ◐ | ◐ | ◐ | Deklaracje backupów znalezione; brak dowodu okresowego restore. |
| Monitoring/incident response | ◐ | ◐ | ◐ | ◐ | Konfiguracje przejrzane; bez logów i dowodu ćwiczeń IR. |
| RODO — dokumentacja | ◐ | ◐ | ◐ | — | Ocena techniczna, nie prawna. |
| RODO — prawa osób | ◐ | ◐ | ◐ | — | Statyczny przegląd endpointów/UI; bez realizacji rzeczywistego wniosku. |
| Aplikacje mobilne | ✅ | ❌ | ✅ | — | Repozytorium Yalquo mobile puste. |
| Provenance/attestations obrazów | ❌ | ❌ | ❌ | ❌ | Brak dostępu do metadanych; nie odpytywano prywatnego registry. |
| Kubernetes Secrets/ConfigMap values/logi | ❌ | ❌ | ❌ | ❌ | Celowo wyłączone ze względu na granice bezpieczeństwa. |

## 4. Skrót architektury i przepływów danych

- Aplikacje web/mobile komunikują się z backendami przez ingress HTTPS.
- Backend Trippics i Yalquo to aplikacje Spring z JWT access/refresh, relacyjnymi bazami danych i magazynem mediów. Yalquo ma również bota operatorskiego.
- Mata24 obejmuje backend Spring, frontend Angular, aplikację Expo/React Native i bota; przetwarza konta rodzinne oraz profile dzieci.
- Argo CD wdraża manifesty ze wspólnego repozytorium GitOps.
- Backupy baz i object storage są planowane przez CronJob i wysyłane do zewnętrznego celu; konfiguracje poświadczeń nie były odczytywane.
- Raport nie publikuje nazw prywatnych hostów, adresów, nazw sekretów, identyfikatorów użytkowników ani surowych danych.

## 5. Priorytetowe findingi

| ID | Severity | Status | Skrót |
|---|---|---|---|
| TRI-SUP-001 | Critical | Confirmed | Eksport bazy Trippics z danymi użytkowników jest śledzony w Git. |
| SHR-K8S-001 | High | Confirmed | Większość workloadów nie ma pełnego hardeningu kontenera. |
| TRI-SEC-002 | Medium | Likely | Upload obrazu nie ma kompletnej walidacji i limitów obciążenia. |
| YAL-RODO-001 | Medium | Confirmed | Publiczny projekt polityki Yalquo nadal ma placeholdery. |
| YAL-RODO-004 | Medium | Confirmed | Profil i część lokalizacji są publiczne domyślnie. |
| SHR-SUP-001 | Medium | Confirmed | Większość obrazów aplikacyjnych nie jest przypięta do digestu. |
| SHR-CICD-001 | Medium | Needs evidence | Brak dowodu ochrony branchy, minimalnych permissions i provenance. |

## 6. Szczegółowe findings

### TRI-SUP-001 — eksport bazy danych z danymi użytkowników w repozytorium

- **Aplikacja i komponent:** Trippics backend, dane migracyjne.
- **Kategoria:** supply chain / sekrety / naruszenie danych.
- **Severity:** Critical. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `trippics-backend/initial-postgres-db-migrated.sql:463-479,40740-40849` jest śledzonym plikiem o rozmiarze około 5,5 MB. Zredagowana analiza potwierdziła 110 rekordów tabeli użytkowników, 110 unikalnych adresów e-mail oraz 83 unikalne fingerprinty wartości o formacie 32-znakowego hasha. W raporcie nie ma żadnej wartości ani danych osoby.
- **Wpływ:** osoby z dostępem do repozytorium i jego historii mogą uzyskać PII oraz materiał do offline cracking; wyciek repozytorium zwiększa ryzyko przejęcia kont i naruszenia danych.
- **Rozwiązanie:** natychmiast ograniczyć dostęp i zachować dowody incydentu; usunąć plik z bieżących refów i całej historii kontrolowaną procedurą; wymusić reset wszystkich objętych haseł i unieważnić powiązane sesje; przeanalizować inne poświadczenia; ocenić obowiązki notyfikacyjne; dodać blokadę dumpów i skan sekretów przed push.
- **Sugerowany właściciel:** Trippics Backend + Security/Incident Response + Privacy.
- **Termin:** natychmiast, ≤24 godziny dla containment; pełna remediacja ≤7 dni.

### SHR-K8S-001 — niepełny hardening kontenerów

- **Aplikacja i komponent:** Trippics, Yalquo, Mata24; workloady Kubernetes.
- **Kategoria:** Kubernetes / workload hardening.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** bezpieczny odczyt live wykazał wyłączenie automatycznego tokenu dla siedmiu głównych workloadów aplikacyjnych, ale tylko jeden z dziewięciu odczytanych workloadów deklarował `runAsNonRoot`, drop `ALL`, `allowPrivilegeEscalation: false` i `seccompProfile`. Pozostałe nie miały kontenerowego `securityContext`; nie znaleziono `readOnlyRootFilesystem` w manifestach aplikacji.
- **Wpływ:** przejęcie procesu aplikacji daje szersze możliwości eskalacji, zapisu w obrazie i nadużycia capabilities.
- **Rozwiązanie:** ustawić na każdym kontenerze `runAsNonRoot`, stały niezerowy UID/GID, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault` i `readOnlyRootFilesystem: true` z jawnie montowanymi katalogami zapisu; egzekwować Pod Security Admission `restricted`.
- **Sugerowany właściciel:** Platform/Kubernetes + zespoły aplikacyjne.
- **Termin:** ≤14 dni dla backendów i botów; ≤30 dni dla całości.

### SHR-K8S-002 — brak bezpiecznego dowodu egzekwowania NetworkPolicy live

- **Aplikacja i komponent:** wspólna infrastruktura, namespace aplikacyjne.
- **Kategoria:** Kubernetes / segmentacja sieci.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** repozytorium GitOps zawiera nowe polityki ingress/egress dla workloadów Trippics, Yalquo i Mata24. API klastra odmówiło jednak listowania `NetworkPolicy`, więc nie potwierdzono ich obecności live ani skuteczności CNI. Nie wykonywano testów połączeń.
- **Wpływ:** błędna synchronizacja lub brak egzekwowania pozostawiłby możliwość niepotrzebnego ruchu lateralnego.
- **Rozwiązanie:** udostępnić audytorowi read-only `get/list` dla NetworkPolicy; potwierdzić zasoby live, zgodność selektorów i działanie default-deny niezależnym, kontrolowanym testem poza tym pasywnym audytem.
- **Sugerowany właściciel:** Platform/Kubernetes.
- **Termin:** evidence ≤7 dni; korekty ≤14 dni.

### TRI-SEC-002 — upload opiera walidację typu na deklaracji klienta

- **Aplikacja i komponent:** Trippics backend, upload zdjęć.
- **Kategoria:** upload plików / walidacja.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** `MeStoryEditorController.java:271-301` przyjmuje typ `image/*`; `FileUploadServiceImpl.java:57-79,105-145` dekoduje i ponownie koduje obraz, ale nie znaleziono kompletnej allow-listy magic bytes, limitu pikseli/decompression ratio ani jawnego limitu pliku w tej ścieżce.
- **Wpływ:** duże lub złośliwie skonstruowane obrazy mogą zużyć pamięć/CPU; błędny typ może zostać propagowany do storage.
- **Rozwiązanie:** wymusić limity request/file i pikseli, wykrywać format po magic bytes, stosować allow-listę dekoderów, timeout i neutralny typ wynikowego pliku.
- **Sugerowany właściciel:** Trippics Backend.
- **Termin:** ≤30 dni.

### TRI-SEC-003 — identyfikatory użytkowników i nazwy plików są logowane

- **Aplikacja i komponent:** Trippics backend, upload/konta.
- **Kategoria:** obsługa błędów / minimalizacja logów.
- **Severity:** Low. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `FileUploadServiceImpl.java:60-61,108-109` oraz `UserServiceImpl.java:173,281,286-291` zapisują identyfikatory/login i nazwy plików w logach. Nie odczytywano logów produkcyjnych.
- **Wpływ:** logi mogą zwiększać zakres danych osobowych i ułatwiać korelację aktywności; nazwy plików mogą zawierać dane użytkownika.
- **Rozwiązanie:** pseudonimizować identyfikatory, nie logować nazw klienta, obniżyć szczegółowość i ustalić krótki TTL oraz dostęp według najmniejszych uprawnień.
- **Sugerowany właściciel:** Trippics Backend + Platform Observability.
- **Termin:** ≤60 dni.

### TRI-RODO-001 — brak kompletnej polityki prywatności

- **Aplikacja i komponent:** Trippics web/mobile/backend.
- **Kategoria:** RODO / przejrzystość.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** w czterech repozytoriach nie znaleziono treści polityki prywatności; znaleziono regulamin i techniczne endpointy usunięcia oraz eksportu (`MeApiController.java:222-260`).
- **Wpływ:** brak spójnej informacji o administratorze, celach, podstawach, retencji, procesorach, transferach, prawach i przetwarzaniu lokalizacji/EXIF.
- **Rozwiązanie:** przygotować zweryfikowaną, wersjonowaną politykę zgodną z rzeczywistymi przepływami i udostępnić ją podczas rejestracji oraz w ustawieniach.
- **Sugerowany właściciel:** Product/Privacy + Trippics Frontend.
- **Termin:** ≤30 dni.

### TRI-RODO-002 — EXIF i dokładne lokalizacje bez jawnej polityki minimalizacji

- **Aplikacja i komponent:** Trippics backend, media/stories.
- **Kategoria:** RODO / privacy by default.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** `FileUploadServiceImpl.java:92,146-176` ekstrahuje EXIF; edytor przyjmuje współrzędne i adres (`MeStoryEditorController.java:318-334`). Nie znaleziono polityki ograniczenia precyzji i retencji.
- **Wpływ:** metadane mogą ujawniać miejsce/czas wykonania zdjęcia i historię podróży.
- **Rozwiązanie:** domyślnie usuwać EXIF z pliku, przechowywać tylko niezbędne pola, ograniczyć precyzję lokalizacji i zapewnić kontrolę widoczności/usunięcia.
- **Sugerowany właściciel:** Trippics Backend/Product Privacy.
- **Termin:** ≤30 dni.

### YAL-RODO-001 — robocza polityka prywatności z placeholderami

- **Aplikacja i komponent:** Yalquo frontend/legal.
- **Kategoria:** RODO / przejrzystość.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `yalquo-frontend/legal/polityka-prywatnosci.md:6-10,98` nadal oznacza dokument jako projekt i zawiera pola `[DO UZUPEŁNIENIA]`. Kopia backendowa jest bardziej kompletna, ale rozbieżność dwóch wersji zwiększa ryzyko publikacji niewłaściwego dokumentu.
- **Wpływ:** obowiązek informacyjny może być niekompletny lub niespójny z dokumentem faktycznie prezentowanym użytkownikowi.
- **Rozwiązanie:** wyznaczyć jedno źródło prawdy, uzupełnić dane administratora, procesorów, transfery i realne retencje, zatwierdzić wersję oraz automatycznie kontrolować zgodność kopii.
- **Sugerowany właściciel:** Product/Privacy Yalquo.
- **Termin:** ≤14 dni.

### YAL-RODO-002 — brak samoobsługowego eksportu danych

- **Aplikacja i komponent:** Yalquo backend/frontend.
- **Kategoria:** RODO / prawa osób.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** dodano chronione hasłem usunięcie konta (`MeApiController.java:98-105`, `UserServiceImpl.java:224-297`) i interfejs frontendu, lecz przegląd kontrolerów nadal nie wykazał uwierzytelnionego eksportu danych.
- **Wpływ:** prawo dostępu i przenoszenia wymaga procesu ręcznego, podatnego na opóźnienia i pominięcia kategorii danych.
- **Rozwiązanie:** wdrożyć bezpieczny eksport maszynowo czytelny, obejmujący profil, aktywność, treści, zgody i metadane, z reautoryzacją, limitem żądań i audytem bez PII.
- **Sugerowany właściciel:** Yalquo Backend/Product Privacy.
- **Termin:** ≤60 dni.

### YAL-RODO-003 — retencja obejmuje tylko część danych bota i jest domyślnie wyłączona

- **Aplikacja i komponent:** Yalquo backend/bot operations.
- **Kategoria:** RODO / retencja.
- **Severity:** Low. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `BotOperationRetentionJob.java:17-37` jest uruchamiany wyłącznie po jawnym włączeniu i usuwa tylko status `SUCCEEDED`; `application.properties:78-81` ustawia `enabled=false`.
- **Wpływ:** wpisy FAILED i inne dane operacyjne mogą być przechowywane bezterminowo, mimo możliwych identyfikatorów użytkowników i payloadów.
- **Rozwiązanie:** ustalić TTL per status/kategoria, włączyć retencję, monitorować skuteczność i udokumentować wyjątki legal hold.
- **Sugerowany właściciel:** Yalquo Backend/Privacy.
- **Termin:** ≤60 dni.

### YAL-RODO-004 — dane profilu i lokalizacja są publiczne domyślnie

- **Aplikacja i komponent:** Yalquo backend/frontend, profile publiczne.
- **Kategoria:** RODO / privacy by default / minimalizacja.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `yalquo-backend/src/main/resources/legal/polityka-prywatnosci.md:95-103` opisuje profil publiczny jako domyślnie włączony, a kraj, region, miasto i odznaki jako domyślnie widoczne; tylko opis „o mnie” jest domyślnie ukryty.
- **Wpływ:** nowi użytkownicy mogą nieświadomie ujawnić lokalizację i aktywność szerszej publiczności, co jest sprzeczne z zasadą ustawień najbardziej chroniących prywatność.
- **Rozwiązanie:** domyślnie wyłączyć profil publiczny i wszystkie opcjonalne pola; zastosować oddzielne, świadome przełączniki, jasny podgląd widoczności i migrację istniejących ustawień po ocenie wpływu.
- **Sugerowany właściciel:** Yalquo Product/Frontend/Privacy.
- **Termin:** ≤30 dni.

### MAT-RODO-002 — brak dowodu DPIA dla przetwarzania danych dzieci

- **Aplikacja i komponent:** Mata24 web/mobile/backend.
- **Kategoria:** RODO / DPIA / dzieci.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** `mata24-frontend/src/app/terms/privacypolicy/privacy.component.html:136-138` opisuje usługę dla dzieci i profile rodzinne. Repozytoria nie zawierają DPIA ani referencji do zatwierdzonej oceny.
- **Wpływ:** ryzyka dla małoletnich, profilowania edukacyjnego i uprawnień rodzic/dziecko mogą nie być formalnie ocenione.
- **Rozwiązanie:** przedstawić lub przeprowadzić DPIA poza publicznym repozytorium, obejmując mapę danych, zagrożenia, środki, konsultację DPO i cykl przeglądu.
- **Sugerowany właściciel:** Data Protection/Privacy + Product Mata24.
- **Termin:** evidence ≤30 dni; DPIA ≤60 dni.

### MAT-RODO-003 — brak dowodu usuwania danych z backupów

- **Aplikacja i komponent:** Mata24 i wspólna infrastruktura backupowa.
- **Kategoria:** RODO / retencja i backupy.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** istnieją cykliczne CronJob backupów, lecz nie znaleziono polityki odnoszącej żądania usunięcia do cyklu kopii ani mechanizmu ponownego zastosowania usunięć po restore.
- **Wpływ:** dane usunięte z produkcji mogą wrócić po odtworzeniu albo pozostać w kopiach dłużej niż zadeklarowano.
- **Rozwiązanie:** określić maksymalny TTL, re-delete ledger po restore i udokumentowany test odtworzenia z kontrolą usuniętych rekordów.
- **Sugerowany właściciel:** Platform/DBA + Privacy.
- **Termin:** ≤60 dni.

### SHR-SUP-001 — większość obrazów nie jest przypięta do immutable digestów

- **Aplikacja i komponent:** wszystkie aplikacje, manifesty GitOps.
- **Kategoria:** supply chain / integralność obrazów.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `apps/mata24-bot/cronjob.yaml:35-37` używa digestu; siedem pozostałych głównych workloadów aplikacyjnych używa tagów będących skróconymi SHA. Odczyt live potwierdził tę samą klasę referencji.
- **Wpływ:** tag może zostać nadpisany; skrócony SHA nie zapewnia kryptograficznego związania manifestu z treścią obrazu.
- **Rozwiązanie:** publikować digest po buildzie, automatycznie aktualizować GitOps i wymuszać digest admission policy; zachować tag wyłącznie jako metadane.
- **Sugerowany właściciel:** Platform/CI.
- **Termin:** ≤30 dni.

### SHR-SUP-002 — obrazy bazowe i narzędziowe CI używają zmiennych tagów

- **Aplikacja i komponent:** Dockerfile i Jenkins pipelines wszystkich aplikacji.
- **Kategoria:** supply chain / CI.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** Dockerfile używają m.in. tagów głównych wersji środowisk Java/Node/nginx, a Jenkinsfile tagów takich jak `debug` i `latest`; nie znaleziono digestów dla tych obrazów.
- **Wpływ:** ponowne wykonanie tego samego commita może pobrać inną warstwę, a kompromitacja lub nadpisanie tagu może wpłynąć na build.
- **Rozwiązanie:** przypiąć obrazy build/runtime do digestów, wdrożyć kontrolowany bot aktualizacyjny, SBOM i podpisane attestations; oddzielić okresową aktualizację od buildów aplikacji.
- **Sugerowany właściciel:** Platform/CI + zespoły aplikacyjne.
- **Termin:** ≤30 dni.

### SHR-CICD-001 — brak dowodu ustawień GitHub i provenance

- **Aplikacja i komponent:** wszystkie repozytoria i pipeline obrazu.
- **Kategoria:** CI/CD / GitHub governance.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** API dla branch protection, rulesets, Actions permissions, workflowów, vulnerability alerts, automated security fixes i Dependabot alerts zwracało `403` dla wszystkich 14 audytowanych repozytoriów. Nie znaleziono repozytoryjnych GitHub Actions; nie uzyskano attestations obrazów.
- **Wpływ:** nie można potwierdzić review, wymaganych kontroli, minimalnych permissions, ochrony branchy ani provenance artefaktów.
- **Rozwiązanie:** zapewnić token audytowy read-only dla tych endpointów; wdrożyć ruleset z review i status checks, ograniczyć uprawnienia botów CI oraz publikować SLSA provenance/SBOM.
- **Sugerowany właściciel:** GitHub Organization Admin + Platform Security.
- **Termin:** evidence ≤14 dni; remediacja ≤30 dni.

### SHR-K8S-003 — brak dowodu testów odtwarzania backupów

- **Aplikacja i komponent:** bazy i storage wszystkich aplikacji.
- **Kategoria:** backup / odporność.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** manifesty deklarują harmonogramy i retencję backupów, ale nie znaleziono protokołu restore, wyniku ostatniego testu ani pomiaru RPO/RTO.
- **Wpływ:** kopia może okazać się niekompletna lub nieodtwarzalna podczas incydentu.
- **Rozwiązanie:** wykonywać okresowy restore do izolowanego środowiska, automatycznie weryfikować integralność i dokumentować RPO/RTO bez danych produkcyjnych.
- **Sugerowany właściciel:** Platform/DBA.
- **Termin:** pierwszy dowód ≤30 dni; cyklicznie co kwartał.

### SHR-SEC-001 — brak jawnego rate limitingu krytycznych endpointów

- **Aplikacja i komponent:** backendy Trippics, Yalquo i Mata24.
- **Kategoria:** abuse prevention / uwierzytelnianie.
- **Severity:** Low. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** nie znaleziono aplikacyjnego limitera dla logowania, resetu hasła, rejestracji, komentarzy i uploadu. Nie odczytywano pełnej konfiguracji kontrolera ingress ani zewnętrznej warstwy ochronnej.
- **Wpływ:** zwiększone ryzyko credential stuffing, enumeracji, spamu i kosztownego przetwarzania.
- **Rozwiązanie:** wdrożyć limity per IP/konto/endpoint, progresywne opóźnienia, limity uploadu i bezpieczne metryki nadużyć; unikać trwałej blokady konta.
- **Sugerowany właściciel:** Backend teams + Platform/Ingress.
- **Termin:** ≤30 dni.

### MAT-K8S-001 — chwilowy drift aplikacji backendowej Mata24

- **Aplikacja i komponent:** Mata24 backend, Argo CD.
- **Kategoria:** Kubernetes / GitOps drift.
- **Severity:** Info. **Confidence:** High. **Status:** Needs evidence.
- **Dowód:** podczas pierwszego odczytu aplikacja była `OutOfSync/Progressing`, a podczas ponownego `OutOfSync/Healthy`; wskazywała aktualną rewizję GitOps. Pozostałe osiem objętych aplikacji było `Synced/Healthy`. Nie pobierano różnicy zawierającej potencjalnie prywatne szczegóły.
- **Wpływ:** stan live może przejściowo lub trwale różnić się od deklaracji, co utrudnia wiarygodną ocenę hardeningu.
- **Rozwiązanie:** po zakończeniu operacji ponownie odczytać status i bezpiecznie sklasyfikować pola driftu; zbadać powtarzający się drift bez ręcznego patchowania zasobów.
- **Sugerowany właściciel:** Platform/GitOps + Mata24 Backend.
- **Termin:** ponowna weryfikacja ≤24 godziny.

## 7. Ocena RODO według aplikacji

### 7.1 Trippics

| Kontrola | Ocena | Dowód/uwaga |
|---|---|---|
| Inwentaryzacja danych, cele i podstawy | ❌ Needs evidence | Brak kompletnej polityki i formalnej mapy danych. |
| Zgody i wycofanie | ◐ | Znaleziono ustawienia komunikacji; brak pełnego rejestru podstaw i wycofań. |
| Przejrzystość/privacy policy | ❌ | TRI-RODO-001. |
| Cookies/analytics/tracking | ◐ | Brak kompletnego publicznego opisu technicznego. |
| Minimalizacja/retencja | ❌ | EXIF/lokalizacja oraz dump bazy; brak harmonogramu wszystkich kategorii. |
| Dostęp/poprawienie/przenoszenie/usunięcie | ◐ | Endpointy eksportu i anonimizacji istnieją; bez testu realizacji i backup lifecycle. |
| Sprzeciw/ograniczenie | ❌ Needs evidence | Brak bezpiecznego dowodu kompletnego procesu. |
| Procesorzy/transfery | ❌ Needs evidence | Brak kompletnej polityki i rejestru procesorów. |
| Privacy by design/default | ❌ | TRI-SUP-001 i TRI-RODO-002. |
| Szyfrowanie/pseudonimizacja | ◐ | TLS ingress i BCrypt dla nowych hashy; historyczne wartości MD5 i PII w dumpie. |
| Naruszenia/DPIA | ❌ Needs evidence | Brak planu naruszeń i DPIA w dostępnych dowodach. |
| Mobile permissions/device IDs | ◐ | Statyczny przegląd konfiguracji; bez store declarations i runtime. |
| Usunięcie konta | ◐ | Endpoint istnieje; brak dowodu usunięcia z kopii. |

### 7.2 Yalquo

| Kontrola | Ocena | Dowód/uwaga |
|---|---|---|
| Inwentaryzacja danych, cele i podstawy | ◐ | Dokumenty opisują wiele kategorii, lecz wersje są niespójne. |
| Zgody i wycofanie | ✅ statycznie | Wersjonowane dokumenty i append-only consent events. |
| Przejrzystość/privacy policy | ❌ | YAL-RODO-001. |
| Cookies/analytics/tracking | ◐ | Opis w dokumentach; brak dynamicznej analizy strony. |
| Minimalizacja/retencja | ❌ | YAL-RODO-003 i publiczne ustawienia lokalizacji. |
| Dostęp/poprawienie/przenoszenie/usunięcie | ◐ | Usunięcie dodane; eksport nadal brak. |
| Sprzeciw/ograniczenie | ◐ | Ustawienia prywatności istnieją; brak pełnego workflow DSAR. |
| Procesorzy/transfery | ❌ Needs evidence | Placeholdery i brak zweryfikowanej listy umów/transferów. |
| Privacy by design/default | ❌ | YAL-RODO-004. |
| Szyfrowanie/pseudonimizacja | ◐ | TLS i anonimizacja konta; brak dowodu szyfrowania danych at rest. |
| Naruszenia/DPIA | ❌ Needs evidence | Brak bezpiecznego dowodu procesu i oceny ryzyka. |
| Mobile permissions/device IDs | ❌ | Repozytorium mobile puste. |
| Usunięcie konta | ✅ statycznie | Reautoryzacja hasłem, revocation refresh tokenów, anonimizacja PII. |

### 7.3 Mata24

| Kontrola | Ocena | Dowód/uwaga |
|---|---|---|
| Inwentaryzacja danych, cele i podstawy | ◐ | Polityka obejmuje konta rodzinne, dane edukacyjne i telemetrykę. |
| Zgody i wycofanie | ◐ | Zgoda opiekuna opisana; brak testu technicznego lifecycle. |
| Przejrzystość/privacy policy | ◐ | Rozszerzona sekcja mobile; wymaga przeglądu prawnego. |
| Cookies/analytics/tracking | ◐ | Brak zewnętrznych SDK crash-reporting; bez dynamicznego skanu cookies. |
| Minimalizacja/retencja | ◐ | Crash reports zredagowane i TTL 30 dni; brak całościowego harmonogramu. |
| Dostęp/poprawienie/przenoszenie/usunięcie | ◐ | Funkcje istnieją częściowo; brak end-to-end DSAR i backup lifecycle. |
| Sprzeciw/ograniczenie | ❌ Needs evidence | Brak kompletnego procesu w dostępnych dowodach. |
| Procesorzy/transfery | ◐ | Część dostawców opisana; umowy i transfer assessment niedostępne. |
| Privacy by design/default | ◐ | Profile dzieci ograniczają e-mail; brak DPIA. |
| Szyfrowanie/pseudonimizacja | ◐ | TLS i secure mobile storage deklarowane; brak dowodu at-rest. |
| Usuwanie z backupów | ❌ Needs evidence | MAT-RODO-003. |
| Naruszenia/DPIA | ❌ Needs evidence | MAT-RODO-002 i brak dowodu ćwiczeń naruszenia. |
| Mobile permissions/device IDs | ◐ | Camera/gallery na żądanie; brak dynamicznej weryfikacji i store declarations. |
| Usunięcie konta | ◐ | Link mobile i mechanizm web; bez testu oraz kopii. |

## 8. Infrastruktura wspólna

### Pozytywne obserwacje

- Wszystkie odczytane ingressy aplikacyjne deklarowały TLS.
- Wszystkie główne kontenery miały requests i limits.
- Backend i frontend głównych aplikacji miały readiness; backendy miały liveness i startup probes.
- Automatyczny token ServiceAccount jest wyłączony w deklaracjach głównych workloadów aplikacyjnych.
- Manifesty NetworkPolicy obejmują ingress i egress dla aplikacji.
- Jeden workload jest przypięty do immutable digestu.
- Osiem z dziewięciu aplikacji Argo CD było `Synced/Healthy`.

### Otwarte obszary

- Pełny `securityContext` nie jest standardem.
- Brak live read-only dowodu NetworkPolicy i RBAC.
- Obrazy i narzędzia CI przeważnie używają tagów.
- Brak dowodu restore, provenance i egzekwowania polityk supply chain.
- Część dodatkowych workloadów w namespace sandbox nie wyłącza tokenu ServiceAccount; ich związek z Yalquo nie został bezpiecznie potwierdzony, więc zapisano to jako lukę pokrycia, nie osobny finding.

## 9. Plan działań

### Natychmiast / 0–7 dni

1. Obsłużyć TRI-SUP-001 jako incydent: containment, historia Git, reset haseł/sesji, ocena naruszenia.
2. Ponownie sprawdzić status Argo CD dla MAT-K8S-001.
3. Udostępnić read-only dowód NetworkPolicy/RBAC i GitHub security settings.
4. Ujednolicić i zablokować publikację roboczej polityki Yalquo.
5. Potwierdzić, że poprzednia remediacja MAT-SUP-001 objęła wszystkie refy i rotację poświadczeń.

### 30 dni

1. Wdrożyć pełny `securityContext` i politykę admission.
2. Przypiąć obrazy runtime/CI do digestów i publikować SBOM/provenance.
3. Naprawić upload Trippics i domyślną prywatność Yalquo.
4. Opublikować politykę prywatności Trippics.
5. Dodać rate limiting krytycznych endpointów.
6. Wykonać i udokumentować pierwszy test restore.

### 60 dni

1. Dodać eksport danych Yalquo i pełne workflow DSAR.
2. Wdrożyć pełne harmonogramy retencji wszystkich aplikacji oraz backup lifecycle.
3. Zakończyć DPIA Mata24 i przegląd procesorów/transferów.
4. Ograniczyć PII w logach i udokumentować retencję observability.

### 90 dni

1. Zautomatyzować ciągłe SAST/SCA/secret scanning bez wykonywania kodu z niezaufanych zmian.
2. Egzekwować rulesets, podpisane artefakty, attestations i admission policies.
3. Przeprowadzić ćwiczenie incident response i odtworzenia danych z mierzalnym RPO/RTO.
4. Powtórzyć pasywny audyt oraz zaplanować odrębny, jawnie autoryzowany test dynamiczny.

## 10. Instrukcje weryfikacji poprawek

- **TRI-SUP-001:** sprawdzić wszystkie refy narzędziem historii Git, potwierdzić brak pliku i fingerprintów; zweryfikować zakończenie resetu haseł/sesji oraz dokument decyzji o notyfikacji. Nie umieszczać danych w ticketach.
- **SHR-K8S-001:** odczytać live pod templates i potwierdzić wszystkie pola `securityContext`; uruchomić kontrolowany test wdrożenia w środowisku nieprodukcyjnym.
- **SHR-K8S-002:** `kubectl get networkpolicy` z read-only rolą; porównać selektory z pod labels, a egzekwowanie sprawdzić osobnym zatwierdzonym testem połączeń.
- **TRI-SEC-002:** testy jednostkowe/integracyjne dla fałszywego MIME, magic bytes, limitu rozmiaru, liczby pikseli i decompression bomb.
- **YAL-RODO-001/004:** potwierdzić jedną wersję dokumentu, brak placeholderów i prywatne ustawienia nowego konta.
- **YAL-RODO-002:** wyeksportować konto testowe i porównać wszystkie kategorie z mapą danych; sprawdzić reautoryzację i czas wygaśnięcia pliku.
- **MAT-RODO-002/003:** przedstawić datowany, zatwierdzony DPIA i protokół restore z kontrolą re-delete.
- **SHR-SUP-001/002:** potwierdzić referencje `@sha256`, zgodność digestu z attestacją i odrzucenie tag-only przez policy engine.
- **SHR-CICD-001:** odczytać rulesets/branch protection, permissions, Dependabot i attestations tokenem tylko do odczytu.
- **MAT-K8S-001:** ponownie odczytać Argo CD po zakończeniu synchronizacji; eskalować, jeśli drift się utrzymuje.

## 11. Coverage gaps

1. GitHub API zwracało `403` dla branch protection, rulesets, Actions permissions/workflows, Dependabot alerts, vulnerability alerts i automated security fixes.
2. Brak dostępu do metadanych attestations/provenance prywatnych obrazów.
3. Kubernetes RBAC odmówił odczytu NetworkPolicy, ServiceAccount, Role i RoleBinding; nie próbowano eskalacji.
4. Nie odczytywano Secrets, wartości ConfigMap, logów, kubeconfigu ani tokenów ServiceAccount.
5. Nie wykonywano DAST, exploitów, prób logowania, uploadu, fuzzingu, port scanningu ani testów połączeń sieciowych.
6. Nie uruchamiano kodu, testów, buildów, package managers, lifecycle scripts ani kontenerów projektu.
7. Brak lokalnej bazy CVE/OSV i narzędzi SCA/SBOM/SAST; zależności oceniono tylko strukturalnie.
8. Nie zweryfikowano dynamicznie cookies, consent bannerów, sklepów mobilnych, push permissions ani deklaracji App Store/Google Play.
9. Nie uzyskano dokumentów organizacyjnych: RoPA, DPIA, DPA/SCC/TIA, procedur DSAR, naruszeń, retencji, legal hold i testów restore.
10. Repozytoria `yalquo-mobile` i `mata24-marketing` są puste.
11. Nie potwierdzono szyfrowania danych i backupów at rest bez odczytu wrażliwej konfiguracji.
12. Drift Argo CD Mata24 nie został rozwinięty do surowej różnicy, aby nie publikować prywatnych szczegółów infrastruktury.

## 12. Appendix — bezpieczne metadane

### Audytowane commity domyślnych gałęzi

| Repozytorium | Branch | Commit SHA |
|---|---|---|
| trippics | main | `f54bad5647a3494eccb3bc16b8a77238a976437c` |
| trippics-backend | main | `5bc237976e0e50d0511064e548703bdf0c385e29` |
| trippics-frontend | main | `8b41902a6f725f9e539472ccabd737db3ba73a7e` |
| trippics-mobile | main | `3f1e7a5eda914e06af64485d71862577d899e7d8` |
| yalquo-backend | main | `2c2568d4b2c0c847e3889c32ef5e5b7f0ffb34da` |
| yalquo-frontend | main | `a671c3f1f0499a27dda161f9321a2bb001cff63c` |
| yalquo-bot | main | `dd12ef56decd731ad2026290b994fdb73baae0a5` |
| matematicon | main | `3ac6299e89b57f194aec5591568750a7fc5bf875` |
| mata24-frontend | main | `61eba0723150ef2d172be001c40fb6c16546ca7c` |
| mata24-mobile | main | `46ee8f84d2a1f386e05cd86082695b3b1071cf8a` |
| mata24-bot | main | `47dbdce4da6ff5ac06c8f81782b810448f8cec0c` |
| local-kubernetes-cluster-definition | main | `41c23134e48f48674cffe60e48b2277169688a86` |
| yalquo-mobile | — | puste repozytorium |
| mata24-marketing | — | puste repozytorium |

### Wersje użytych narzędzi

- `git 2.47.3`
- `gh 2.96.0`
- `kubectl 1.28.15` (client)
- `Python 3.13.14` standard library — lokalne, heurystyczne skrypty tylko do odczytu i agregacji.
- Dedykowane narzędzia SAST/SCA/secret scanning nie były dostępne; nie instalowano zależności.

### Obrazy — publicznie bezpieczne referencje

- Siedem głównych workloadów aplikacyjnych: prywatny registry zredagowany; referencja klasy `tag`, zwykle 8-znakowy prefiks commit SHA.
- Jeden workload aplikacyjny: prywatny registry i digest zredagowane; referencja klasy `immutable digest`.
- Obrazy bazowe obejmują rodziny Eclipse Temurin, Node, nginx, Alpine i Kaniko; używane są tagi, nie digesty.
- Nie publikowano prywatnych nazw hostów registry, pełnych digestów ani credentiali.
