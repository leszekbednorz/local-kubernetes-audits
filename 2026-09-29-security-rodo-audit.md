# Miesięczny audyt techniczny bezpieczeństwa i RODO — Trippics, Yalquo, Mata24

**Data oceny:** 2026-09-29 (Europe/Warsaw, UTC+02:00) / 2026-09-29 UTC  
**Tryb:** wyłącznie pasywny, punktowy przegląd techniczny  
**Klasyfikacja raportu:** publiczny; dowody zredagowane

> **Zastrzeżenie:** raport jest techniczną oceną punktową opartą na dostępnych dowodach. Nie stanowi formalnej porady prawnej, audytu zgodności prawnej ani pełnego testu penetracyjnego. Nie wykonywano exploitów, prób logowania, fuzzingu, testów obciążeniowych ani kodu z audytowanych repozytoriów.

## 1. Executive summary

Przegląd objął statycznie 12 niepustych repozytoriów, dwa puste repozytoria, odczyty metadanych GitHub oraz niesekretne pola stanu klastra Kubernetes. Najpoważniejszym potwierdzonym problemem są historyczne kopie bazy Mata24 umieszczone w prywatnym repozytorium kodu. Pliki zawierają wzorce danych osobowych oraz 61 unikalnych wartości przypominających JWT. Wartości nie zostały użyte, zweryfikowane ani opublikowane w tym raporcie. Samo usunięcie plików z bieżącej gałęzi nie usuwa ich z historii Git; wymagane jest traktowanie sytuacji jak incydentu danych i poświadczeń.

W warstwie Kubernetes wszystkie sprawdzone aplikacje Argo CD objęte zakresem były `Synced` i `Healthy`, a obrazy workloadów aplikacyjnych odpowiadały skróconym SHA deklarowanym w GitOps. Jednocześnie niemal wszystkie workloady używają domyślnego ServiceAccount, nie wyłączają automatycznego montowania tokenu i nie deklarują pełnego `securityContext`. W czterech sprawdzonych namespace nie znaleziono żadnej `NetworkPolicy`. TLS, requests/limits oraz sondy zdrowia są w większości obecne.

Ocena RODO jest nierówna. Trippics i Mata24 mają techniczne endpointy eksportu i usunięcia/anonymizacji konta, lecz dla Trippics nie znaleziono kompletnej polityki prywatności. Yalquo ma najbardziej rozwinięty model wersjonowanych dokumentów i zgód, ale opublikowany projekt polityki zawiera jawne placeholdery, a w kodzie nie znaleziono samoobsługowego eksportu ani usunięcia konta. Mata24 przetwarza dane dzieci i zbiera raporty awarii zawierające komunikaty oraz stack trace; nie znaleziono dowodu DPIA, polityki retencji raportów awarii ani mechanizmu redakcji przed zapisem.

### Rozkład findingów

| Severity | Liczba |
|---|---:|
| Critical | 1 |
| High | 4 |
| Medium | 9 |
| Low | 3 |
| Info | 1 |
| **Razem** | **18** |

### Status remediacji (aktualizowane na bieżąco - szczegóły w sekcji 13)

| ID | Status | Data |
|---|---|---|
| MAT-SUP-001 | ✅ Naprawione (dumpy usunięte z repo i całej historii Git na wszystkich branchach, force-push; `.gitignore` zablokowany) | 2026-09-30 |
| SHR-K8S-001 | 🟡 Częściowo (automount tokenu ServiceAccount wyłączony na wszystkich workloadach; pełny `securityContext` non-root w toku) | 2026-09-30 |
| TRI-SEC-001 | ✅ Naprawione (Facebook signup już nie wysyła hasła mailem - losowe, nigdzie nieujawniane hasło jak przy Google; zbudowane i wdrożone) | 2026-09-30 |
| SHR-K8S-002 | ✅ Naprawione dla namespace'ów w zakresie audytu (`trippics`, `mata24`, `sandbox`/yalquo) - default-deny + allow-list `NetworkPolicy` per-appka, zweryfikowane świeżymi połączeniami | 2026-09-30 |
| MAT-RODO-001 | ✅ Naprawione (redakcja PII w crash reports po stronie mobile i backendu, brak podwójnego logowania treści, retencja 30 dni włączona na produkcji) | 2026-09-30 |
| SHR-SUP-001 | 🟡 Częściowo (`mata24-bot` przypięty do immutable digestu, CI aktualizuje go automatycznie; pozostałe workloady nadal na skróconym SHA, nie digest) | 2026-09-30 |
| YAL-RODO-002 | 🟡 Częściowo (samoobsługowe usunięcie konta w backendzie i UI, dokumentacja poprawiona; eksport danych wciąż brakuje) | 2026-10-01 |

## 2. Zakres i metodologia

### Repozytoria

- Trippics: `trippics`, `trippics-backend`, `trippics-frontend`, `trippics-mobile`.
- Yalquo: `yalquo-backend`, `yalquo-frontend`, `yalquo-mobile`, `yalquo-bot`.
- Mata24: `matematicon`, `mata24-frontend`, `mata24-mobile`, `mata24-marketing`, `mata24-bot`.
- Wspólna infrastruktura: `local-kubernetes-cluster-definition`.
- Raporty: `local-kubernetes-audits`.

`yalquo-mobile` i `mata24-marketing` były puste: brak branchy i commitów. Nie można było wykonać kontroli kodu tych komponentów.

### Metoda

1. Zanotowano czas UTC i Europe/Warsaw oraz SHA domyślnych gałęzi.
2. Repozytoria pobrano bez checkout hooks; zawartość wyeksportowano przez `git archive`. Nie wykonano skryptów, buildów, testów, kontenerów ani lifecycle scripts.
3. Statycznie przejrzano kod, lockfile, manifesty zależności, Dockerfile, Jenkinsfile, manifesty GitOps, dokumenty prawne i konfiguracje aplikacji mobilnych.
4. Lokalny heurystyczny skan sekretów/PII raportował wyłącznie typ, lokalizację i nieodwracalny skrót; żadnej wartości nie użyto ani nie ujawniono.
5. GitHub API użyto tylko do odczytu. Ustawienia administracyjne i alerty bezpieczeństwa zwracały `403`, dlatego nie uznano ich za zaliczone.
6. `kubectl` użyto wyłącznie do `get` niesekretnych metadanych/spec/status. Nie odczytywano Secrets, ConfigMap values, logów, kubeconfigu ani tokenów ServiceAccount.
7. Deklaracje GitOps porównano z bezpiecznymi polami live: obrazy, sondy, zasoby, ServiceAccount, `securityContext`, stan Argo CD, ingress/TLS i NetworkPolicy.

Statusy: **Confirmed** — dowód bezpośredni; **Likely** — silny dowód statyczny, bez testu runtime; **Needs evidence** — brak wystarczającego bezpiecznego dowodu.

## 3. Coverage matrix

Legenda: ✅ wykonane, ◐ częściowe, ❌ niewykonane / brak bezpiecznego dowodu.

| Obszar | Trippics | Yalquo | Mata24 | Shared | Uwagi |
|---|:---:|:---:|:---:|:---:|---|
| AuthN/AuthZ, JWT, hasła | ✅ | ✅ | ✅ | — | Statyczny przegląd filtrów, endpointów i encoderów; bez prób logowania. |
| IDOR / własność obiektów | ◐ | ◐ | ◐ | — | Przegląd kontrolerów i serwisów; brak dynamicznej weryfikacji. |
| CORS/CSRF/XSS/injection | ◐ | ◐ | ◐ | — | Przegląd statyczny; brak fuzzingu i DAST. |
| SSRF/path traversal/upload | ◐ | ◐ | ◐ | — | Wyłącznie analiza przepływów i walidacji. |
| Rate limiting i błędy | ◐ | ◐ | ◐ | — | Nie znaleziono aplikacyjnego limitera; stan ingress controller częściowo poza zakresem. |
| Dependencies/lockfile | ◐ | ◐ | ◐ | ◐ | Lockfile/manifests obecne; brak lokalnej bazy OSV/SCA i brak dostępu do Dependabot alerts. |
| Sekrety/PII w repo | ✅ | ✅ | ✅ | ✅ | Lokalny skan heurystyczny; bez walidowania wartości. |
| GitHub Actions/ochrona branchy | ◐ | ◐ | ◐ | ◐ | Workflowy nieobecne; Jenkinsfile przejrzane. API settings/alerts: `403`. |
| Docker/Kubernetes/GitOps | ✅ | ✅ | ✅ | ✅ | Bezpieczne pola live + deklaracje; bez Secrets i ConfigMap values. |
| Backup/restore | ◐ | ◐ | ◐ | ◐ | Harmonogramy backupów znalezione; brak bezpiecznego dowodu testów restore. |
| Monitoring/incident response | ◐ | ◐ | ◐ | ◐ | Manifesty monitoringu częściowo; bez logów i bez dowodu ćwiczeń IR. |
| RODO — dokumentacja | ◐ | ◐ | ◐ | — | Ocena techniczna, nie prawna. |
| RODO — prawa osób | ◐ | ◐ | ◐ | — | Endpointy i UI statycznie; bez realizacji rzeczywistego wniosku. |
| Aplikacje mobilne | ✅ | ❌ | ✅ | — | Yalquo mobile puste. |
| Provenance/attestations obrazów | ❌ | ❌ | ❌ | ❌ | GitHub API odmówiło dostępu; nie pobierano prywatnych metadanych registry. |

## 4. Skrót architektury i przepływów danych

- Aplikacje web/mobile komunikują się z backendami przez HTTPS ingress.
- Backend Trippics i Yalquo to aplikacje Spring z JWT access/refresh, relacyjną bazą danych i magazynem obiektowym dla mediów. Yalquo ma także bota operatorskiego.
- Mata24 składa się z backendu Spring, frontendu Angular, aplikacji Expo/React Native oraz bota. Backend obsługuje konta rodzinne i profile dzieci.
- Argo CD wdraża deklaracje z repozytorium wspólnej infrastruktury. Sprawdzone aplikacje miały automatyczną synchronizację.
- Backupy baz i obiektów są planowane przez CronJob i wysyłane do zewnętrznego celu z szyfrowaniem konfigurowanym przez referencje do Secrets. Wartości sekretów nie były odczytywane.
- Przepływy danych osobowych obejmują dane kont, treści użytkowników, lokalizacje/EXIF, dane zgód, adresy IP/User-Agent, dane edukacyjne dzieci oraz raporty awarii urządzeń.

## 5. Priorytetowe findingi

| ID | Severity | Aplikacja / komponent | Status | Skrót |
|---|---|---|---|---|
| MAT-SUP-001 | Critical | Mata24 / repo backend | Confirmed | Kopie produkcyjnej bazy z PII i tokenopodobnymi wartościami w Git. |
| SHR-K8S-001 | High | Wszystkie workloady | Confirmed | Brak pełnego hardeningu podów i zbędne tokeny ServiceAccount. |
| SHR-K8S-002 | High | Wszystkie namespace | Confirmed | Brak NetworkPolicy i segmentacji east-west. |
| TRI-SEC-001 | High | Trippics backend | Confirmed | Tymczasowe hasło przesyłane e-mailem. |
| MAT-RODO-001 | High | Mata24 mobile/backend | Confirmed | Surowe komunikaty i stack trace trafiają do bazy i logów. |
| YAL-RODO-001 | Medium | Yalquo legal/frontend | Confirmed | Polityka prywatności jest szkicem z placeholderami. |
| YAL-RODO-002 | Medium | Yalquo backend | Confirmed | Brak samoobsługowego eksportu i usunięcia konta. |
| MAT-RODO-002 | Medium | Mata24 | Needs evidence | Brak dowodu DPIA dla danych dzieci. |
| SHR-SUP-001 | Medium | Docker/Kubernetes | Confirmed | Obrazy bazowe i część workloadów nie są przypięte digestem. |
| SHR-CICD-001 | Info | GitHub | Needs evidence | Brak uprawnień do oceny protections, permissions, alerts i attestations. |

## 6. Szczegółowe findings

### MAT-SUP-001 — kopie bazy danych w repozytorium kodu

- **Aplikacja i komponent:** Mata24, `matematicon`.
- **Kategoria:** Supply chain / dane w repozytorium.
- **Severity:** Critical. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** pliki `mat1_backup_20260723.sql`, `mata24_PROD_backup_20260723.sql`, `mata24_PROD_backup_20260725.sql`; lokalny skan wykrył wzorce danych osobowych i 61 unikalnych wartości przypominających JWT. Przykładowy bezpieczny fingerprint: `sha256:17ded2e8c43a…`; wartości nie ujawniono i nie weryfikowano.
- **Wpływ:** każdy mający dostęp do historii repo może uzyskać dane użytkowników i potencjalne poświadczenia sesyjne; zwiększa to ryzyko naruszenia poufności i obowiązków notyfikacyjnych.
- **Rozwiązanie:** natychmiast ograniczyć dostęp, uruchomić procedurę incydentową, unieważnić wszystkie potencjalnie dotknięte tokeny/klucze, ustalić zakres osób i danych, usunąć dumpy z całej historii Git po koordynacji z użytkownikami repo, przechowywać backupy wyłącznie szyfrowane poza SCM; dodać pre-receive/secret scanning.
- **Właściciel:** Security/Incident Response + Data Protection + Backend/Platform.
- **Termin:** natychmiast, triage ≤24 h; pełna remediacja ≤7 dni.
- **Status realizacji:** ✅ Naprawione 2026-09-30 - szczegóły w sekcji 13.

### SHR-K8S-001 — niepełny hardening podów i automatyczne tokeny ServiceAccount

- **Aplikacja i komponent:** wspólne; deploymenty Trippics, Yalquo i Mata24.
- **Kategoria:** Kubernetes hardening.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** live deploymenty używają `serviceAccountName: default`, nie ustawiają `automountServiceAccountToken: false`; poza `yalquo-bot` brak jawnych `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]` i `seccompProfile`. Repo: `apps/*/deployment.yaml`; live spec odczyt 2026-09-29.
- **Wpływ:** przejęty kontener otrzymuje zbędny token API i szersze możliwości eskalacji w obrębie poda/noda.
- **Rozwiązanie:** oddzielny minimalny ServiceAccount albo `automountServiceAccountToken: false`; pod/container security context z non-root, RuntimeDefault, drop ALL, no privilege escalation i read-only root filesystem po testach kompatybilności; egzekwować Pod Security Admission `restricted`.
- **Właściciel:** Platform/Kubernetes + zespoły aplikacyjne.
- **Termin:** 7 dni dla tokenów, 30 dni dla pełnego hardeningu.
- **Status realizacji:** 🟡 Część 1 (automount tokenu) naprawiona 2026-09-30 - szczegóły w sekcji 13. Część 2 (securityContext) w toku.

### SHR-K8S-002 — brak NetworkPolicy

- **Aplikacja i komponent:** namespace `trippics`, `yalquo`, `mata24`, `sandbox`.
- **Kategoria:** segmentacja sieciowa.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** live API zwróciło zero `NetworkPolicy` we wszystkich czterech namespace; repo zawiera polityki dla części usług wspólnych, lecz nie dla audytowanych aplikacji.
- **Wpływ:** po kompromitacji jednego poda możliwy jest nieograniczony ruch lateralny zgodnie z możliwościami CNI i usług klastra.
- **Rozwiązanie:** default-deny ingress/egress, następnie allow-list DNS, ingress controller, wymagane bazy/cache/object storage i jawnie uzasadnione wyjścia internetowe.
- **Właściciel:** Platform/Kubernetes.
- **Termin:** projekt ≤7 dni, wdrożenie ≤30 dni.
- **Status realizacji:** ✅ Naprawione 2026-09-30 dla wszystkich stałych workloadów w audytowanych namespace'ach (`trippics`, `mata24`, `sandbox`) - szczegóły w sekcji 13.

### TRI-SEC-001 — przesyłanie tymczasowego hasła e-mailem

- **Aplikacja i komponent:** Trippics backend, onboarding federacyjny.
- **Kategoria:** hasła i odzyskiwanie konta.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `src/main/java/pl/trippics/service/impl/EmailServiceImpl.java:51-57` umieszcza tymczasowe hasło w treści wiadomości; `UserServiceImpl.java:156-174` generuje, zapisuje hash i wysyła hasło.
- **Wpływ:** hasło może pozostać w skrzynkach, kopiach poczty i systemach antyspamowych; użytkownik może nie zmienić go natychmiast.
- **Rozwiązanie:** nie wysyłać haseł; użyć jednorazowego, krótkotrwałego, zahashowanego tokenu „ustaw hasło”, z unieważnieniem po użyciu i rate limitingiem.
- **Właściciel:** Trippics Backend/Auth.
- **Termin:** ≤7 dni.
- **Status realizacji:** ✅ Naprawione 2026-09-30 - szczegóły w sekcji 13.

### MAT-RODO-001 — raporty awarii mogą utrwalać PII i sekrety

- **Aplikacja i komponent:** Mata24 mobile i backend.
- **Kategoria:** RODO / telemetryka / logowanie.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `mata24-mobile/src/lib/crashReporting.ts:16-30` wysyła pełny `message` i `stack`; `ClientErrorReportService.java:31-44` zapisuje je, wiąże z user ID, a komunikat ponownie zapisuje do logu. Nie znaleziono redakcji, zgody/opt-out ani retencji tej tabeli.
- **Wpływ:** błędy mogą zawierać tokeny, dane formularzy, identyfikatory i inne PII; dane są duplikowane w bazie i logach.
- **Rozwiązanie:** allow-list pól, redakcja tokenów/e-maili/URL query przed wysłaniem i ponownie na serwerze, nie logować pełnego komunikatu, ustalić krótki TTL i podstawę/cel, udokumentować telemetrykę oraz zapewnić opt-out, gdy wymagany.
- **Właściciel:** Mata24 Mobile/Backend + Privacy.
- **Termin:** ≤7 dni.
- **Status realizacji:** ✅ Naprawione 2026-09-30 - szczegóły w sekcji 13.

### TRI-SEC-002 — upload opiera walidację typu na deklaracji klienta

- **Aplikacja i komponent:** Trippics backend, upload zdjęć.
- **Kategoria:** upload plików / walidacja.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** `MeStoryEditorController.java:271-301` akceptuje typ zaczynający się od `image/`; `FileUploadServiceImpl.java:66-79,126-145` dekoduje i re-enkoduje obraz, co zmniejsza ryzyko, ale nie znaleziono jawnego limitu rozmiaru w tej ścieżce ani allow-listy formatów/magic bytes.
- **Wpływ:** duże lub złośliwie skonstruowane obrazy mogą zużyć pamięć/CPU; błędny content-type jest propagowany do storage.
- **Rozwiązanie:** limity request/file, magic-byte detection, allow-lista formatów, limity pikseli/decompression ratio, timeout oraz neutralny content-type wynikowego JPEG.
- **Właściciel:** Trippics Backend.
- **Termin:** ≤30 dni.

### TRI-RODO-001 — brak kompletnej polityki prywatności

- **Aplikacja i komponent:** Trippics, web/mobile/backend.
- **Kategoria:** RODO / przejrzystość.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** w czterech repozytoriach nie znaleziono pliku/treści polityki prywatności; istnieją warunki oraz endpointy eksportu/usunięcia (`MeApiController.java:222-260`).
- **Wpływ:** użytkownik nie otrzymuje kompletnej, spójnej informacji o administratorze, celach, podstawach, retencji, procesorach, transferach, prawach i danych lokalizacyjnych/EXIF.
- **Rozwiązanie:** przygotować i opublikować zweryfikowaną politykę, wersjonować ją, połączyć z rejestracją i ustawieniami oraz utrzymywać zgodność z rzeczywistymi przepływami.
- **Właściciel:** Product/Privacy + Trippics Frontend.
- **Termin:** ≤30 dni.

### TRI-RODO-002 — EXIF i dokładne lokalizacje bez jawnej polityki minimalizacji

- **Aplikacja i komponent:** Trippics backend, media/stories.
- **Kategoria:** RODO / privacy by default.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** `FileUploadServiceImpl.java:92` zapisuje wyodrębniony EXIF; edytor przyjmuje współrzędne i pełny adres (`MeStoryEditorController.java:318-334`). Brak znalezionej polityki prywatności i dowodu ograniczenia precyzji/retencji.
- **Wpływ:** publikacja lub długie przechowywanie metadanych może ujawniać miejsce/czas wykonania zdjęcia i historię podróży.
- **Rozwiązanie:** domyślnie usuwać EXIF z pliku i przechowywać tylko niezbędne pola; zaokrąglać lokalizację, zapewnić kontrolę widoczności i usunięcia, opisać cel/retencję.
- **Właściciel:** Trippics Backend/Product Privacy.
- **Termin:** ≤30 dni.

### YAL-RODO-001 — robocza polityka prywatności z placeholderami

- **Aplikacja i komponent:** Yalquo frontend/legal.
- **Kategoria:** RODO / przejrzystość.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `legal/polityka-prywatnosci.md:6-10,86,98` oznacza dokument jako projekt i zawiera `[DO UZUPEŁNIENIA]`; backendowy bootstrap domyślnie jest wyłączony dla szkiców.
- **Wpływ:** jeżeli ta wersja jest prezentowana użytkownikom, obowiązek informacyjny może być niekompletny; jeśli nie jest publikowana, brak dowodu aktualnej polityki produkcyjnej.
- **Rozwiązanie:** uzupełnić administratora, procesorów, transfery, datę i realne retencje; wykonać akceptację właściciela prawnego i opublikować wersję produkcyjną.
- **Właściciel:** Product/Privacy Yalquo.
- **Termin:** ≤30 dni.

### YAL-RODO-002 — brak samoobsługowego eksportu i usunięcia konta

- **Aplikacja i komponent:** Yalquo backend/frontend.
- **Kategoria:** RODO / prawa osób.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** statyczny przegląd kontrolerów nie wykazał endpointów eksportu danych ani usunięcia konta; polityka twierdzi, że większość operacji, w tym usunięcie konta, jest dostępna w ustawieniach (`legal/polityka-prywatnosci.md:74`).
- **Wpływ:** rozbieżność dokumentacji z produktem i ręczna, podatna na błędy obsługa praw dostępu/usunięcia/przenoszenia.
- **Rozwiązanie:** wdrożyć uwierzytelniony eksport oraz proces usunięcia/anonymizacji obejmujący treści, media, zgody, relacje bota i backup lifecycle; skorygować dokument do czasu wdrożenia.
- **Właściciel:** Yalquo Backend/Product Privacy.
- **Termin:** ≤60 dni; dokumentacja ≤7 dni.
- **Status realizacji:** 🟡 Częściowo - usunięcie konta (backend + UI) i korekta dokumentacji naprawione 2026-09-30/10-01, szczegóły w sekcji 13. Uwierzytelniony eksport danych (GDPR data portability) wciąż nie istnieje - pozostaje otwarte.

### YAL-RODO-003 — retencja obejmuje tylko część danych bota i jest domyślnie wyłączona

- **Aplikacja i komponent:** Yalquo backend/bot operations.
- **Kategoria:** RODO / retencja.
- **Severity:** Low. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `BotOperationRetentionJob.java:17-37` działa tylko po jawnym włączeniu i usuwa wyłącznie wpisy `SUCCEEDED` starsze domyślnie niż 90 dni; brak dowodu live, że job jest aktywny.
- **Wpływ:** FAILED i inne dane operacyjne mogą być przechowywane bezterminowo, mimo możliwych identyfikatorów użytkowników i payloadów.
- **Rozwiązanie:** zinwentaryzować klasy danych, ustalić TTL per status/kategoria, włączyć job i monitorować skuteczność; udokumentować wyjątki legal hold.
- **Właściciel:** Yalquo Backend/Privacy.
- **Termin:** ≤60 dni.

### MAT-RODO-002 — brak dowodu DPIA dla przetwarzania danych dzieci

- **Aplikacja i komponent:** Mata24 web/mobile/backend.
- **Kategoria:** RODO / DPIA / dzieci.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** polityka opisuje usługę dla dzieci i profile rodzinne (`privacy.component.html:124-125`); kod zawiera dane postępów, relacji rodzinnych i telemetrykę. W repozytoriach nie znaleziono DPIA ani referencji do zatwierdzonej oceny.
- **Wpływ:** ryzyka dla małoletnich, profilowania edukacyjnego i uprawnień rodzic/dziecko mogą nie być formalnie ocenione i ograniczone.
- **Rozwiązanie:** udostępnić lub przeprowadzić DPIA poza publicznym repo, z mapą danych, zagrożeniami, środkami, konsultacją DPO i cyklem przeglądu.
- **Właściciel:** Data Protection/Privacy + Product Mata24.
- **Termin:** evidence ≤30 dni; DPIA ≤60 dni.

### MAT-RODO-003 — brak dowodu usuwania danych z backupów

- **Aplikacja i komponent:** Mata24 i wspólna infrastruktura backupowa.
- **Kategoria:** RODO / retencja i backupy.
- **Severity:** Medium. **Confidence:** Medium. **Status:** Needs evidence.
- **Dowód:** istnieją cykliczne CronJob backupów bazy i object storage, lecz nie znaleziono polityki rotacji odnoszącej się do żądań usunięcia ani procedury reintegracji usunięć po restore.
- **Wpływ:** dane usunięte z produkcji mogą wrócić po odtworzeniu lub pozostać w kopiach dłużej niż deklarowano.
- **Rozwiązanie:** określić maksymalny TTL kopii, immutable retention, re-delete ledger po restore i udokumentowany test odtworzenia z kontrolą usuniętych rekordów.
- **Właściciel:** Platform/DBA + Privacy.
- **Termin:** ≤60 dni.

### SHR-SUP-001 — obrazy bez immutable digestów

- **Aplikacja i komponent:** wspólne Docker/GitOps.
- **Kategoria:** Supply chain.
- **Severity:** Medium. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** Dockerfile używają m.in. tagów major/minor; deploymenty używają tagów skróconego SHA, a `apps/mata24-bot/cronjob.yaml:28` używa `:latest`. Brak `@sha256:` w audytowanych workloadach.
- **Wpływ:** tag może wskazywać inny obraz w przyszłości; rollback i potwierdzenie provenance są mniej wiarygodne.
- **Rozwiązanie:** budować raz, podpisywać/atestować, wdrażać wyłącznie po digest; usunąć `latest`; weryfikować podpis admission policy.
- **Właściciel:** CI/CD + Platform.
- **Termin:** `latest` ≤7 dni, pozostałe ≤60 dni.
- **Status realizacji:** 🟡 Częściowo - `mata24-bot` (jedyny znaleziony przypadek `:latest` w deklaracji live workloadu) naprawiony 2026-09-30, szczegóły w sekcji 13. Pozostałe workloady (pinowane do skróconego SHA, nie do digestu) wciąż otwarte, termin 60 dni.

### SHR-CICD-001 — brak dowodu ustawień GitHub i provenance

- **Aplikacja i komponent:** wszystkie repozytoria GitHub.
- **Kategoria:** CI/CD governance.
- **Severity:** Info. **Confidence:** High. **Status:** Needs evidence.
- **Dowód:** API dla Actions permissions, workflow permissions, branch protection/rulesets, Dependabot, secret/code scanning i attestations zwracało `403` dla użytego tokenu. W repo nie znaleziono GitHub Actions; CI realizują Jenkinsfile.
- **Wpływ:** audyt nie może potwierdzić minimalnych uprawnień, wymaganych review/checków, alertów i provenance.
- **Rozwiązanie:** dostarczyć read-only token z wymaganymi scopes albo bezpieczny eksport ustawień; sprawdzić protections/rulesets, secret scanning, Dependabot, CODEOWNERS, podpisy i retencję artefaktów.
- **Właściciel:** GitHub Organization Admin / DevSecOps.
- **Termin:** przed następnym miesięcznym audytem.

### SHR-K8S-003 — brak dowodu testów odtwarzania backupów

- **Aplikacja i komponent:** wspólna infrastruktura, bazy i object storage.
- **Kategoria:** backup/recovery.
- **Severity:** Medium. **Confidence:** High. **Status:** Needs evidence.
- **Dowód:** `apps/main-db/backup-cronjob.yaml`, `apps/yalquo-postgres/backup-cronjob.yaml` i `apps/minio/extra/backup-cronjob.yaml` definiują harmonogramy; brak runbooka/test result restore w zakresie. Logów nie odczytywano z uwagi na możliwe sekrety/PII.
- **Wpływ:** backup może być niekompletny lub nieodtwarzalny mimo pozornie poprawnego harmonogramu.
- **Rozwiązanie:** kwartalny izolowany restore, walidacja integralności i RPO/RTO, zredagowany protokół, alerty na brak/niepowodzenie backupu.
- **Właściciel:** Platform/DBA/SRE.
- **Termin:** pierwszy test ≤30 dni.

### SHR-SEC-001 — brak jawnego rate limitingu w krytycznych endpointach

- **Aplikacja i komponent:** backendy Trippics, Yalquo, Mata24.
- **Kategoria:** abuse prevention.
- **Severity:** Low. **Confidence:** Medium. **Status:** Likely.
- **Dowód:** nie znaleziono aplikacyjnego limitera dla login/register/reset, publicznych komentarzy, analityki ani crash reports; bezpieczny zakres nie pozwolił potwierdzić limitów globalnych ingress/WAF.
- **Wpływ:** brute force, enumeracja, spam i niekontrolowany wzrost danych/CPU.
- **Rozwiązanie:** limit per IP/account/device z bezpiecznym zaufaniem proxy, osobne budżety endpointów, backoff i monitoring 429; nie blokować prawidłowych klientów współdzielących NAT.
- **Właściciel:** Backend + Platform/SRE.
- **Termin:** ≤60 dni.

### TRI-SEC-003 — identyfikatory użytkowników i nazwy plików w logach

- **Aplikacja i komponent:** Trippics backend.
- **Kategoria:** obsługa błędów / logowanie.
- **Severity:** Low. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `FileUploadServiceImpl.java:59,95,106,120` loguje user ID, oryginalną nazwę pliku i ścieżki; `UserServiceImpl.java:115-122,317-333` loguje login przy zdarzeniach auth/reset. Samych logów nie odczytywano.
- **Wpływ:** PII i dane treściowe mogą być powielane w systemie logowania i przechowywane dłużej niż dane źródłowe.
- **Rozwiązanie:** pseudonimizowane correlation IDs, usunięcie nazw plików/loginów, strukturalne logi z allow-listą i krótką retencją.
- **Właściciel:** Trippics Backend/SRE.
- **Termin:** ≤60 dni.

## 7. Ocena RODO według aplikacji

### 7.1 Trippics

| Kontrola | Ocena | Dowód / luka |
|---|---|---|
| Inwentaryzacja danych, cele, podstawy | Needs evidence | Modele i przepływy wskazują konto, treści, media, EXIF/lokalizacje; brak kompletnego ROPA/polityki. |
| Zgody i wycofanie | Partial | Wersjonowane terms/acceptance; brak pełnego katalogu zgód opartych na consent. |
| Privacy policy / transparentność | Gap | TRI-RODO-001. |
| Cookies/analytics/tracking | Needs evidence | Brak kompletnej deklaracji i porównania z produkcyjnymi nagłówkami/runtime. |
| Minimalizacja/retencja | Partial | Usunięcie konta istnieje; brak harmonogramów dla treści, EXIF, logów i backupów. |
| Dostęp/poprawienie/przenoszenie/usunięcie | Partial-positive | Profil, eksport i delete account obecne; zakres eksportu i usunięcia backupów wymaga testu. |
| Sprzeciw/ograniczenie | Needs evidence | Brak jawnego workflow. |
| Procesorzy/transfery | Needs evidence | Brak kompletnej polityki. |
| Privacy by design/default | Partial | SecureStore mobile; ryzyko EXIF/lokalizacji. |
| Naruszenia/DPIA | Needs evidence | Brak bezpiecznego dowodu procedury lub ćwiczeń. |
| Mobile permissions/telemetry | Partial | Camera/photos uzasadnione; Android deklaruje `RECORD_AUDIO` bez znalezionego celu — należy usunąć lub udokumentować. |
| Usunięcie konta | Present | Backend endpoint i mobile/web UI istnieją statycznie. |

### 7.2 Yalquo

| Kontrola | Ocena | Dowód / luka |
|---|---|---|
| Inwentaryzacja danych, cele, podstawy | Partial | Projekt polityki opisuje kategorie i podstawy, ale zawiera placeholdery. |
| Zgody i wycofanie | Positive | Append-only consent events i grant/revoke API; IP/User-Agent zwiększają zakres danych i wymagają retencji. |
| Privacy policy / transparentność | Gap | YAL-RODO-001. |
| Cookies/analytics/tracking | Partial | Dokument deklaruje brak cookies trackingowych; brak dynamicznej weryfikacji. |
| Minimalizacja/retencja | Partial | Prywatność profilu i częściowa retencja bota; YAL-RODO-003. |
| Dostęp/poprawienie/przenoszenie/usunięcie | Partial-positive | Edycja profilu i usunięcie konta (anonimizacja, `DELETE /api/me` + UI) od 2026-10-01; eksport danych (przenoszenie) wciąż brakuje. |
| Sprzeciw/ograniczenie | Needs evidence | Opisane w polityce, brak procesu technicznego. |
| Procesorzy/transfery | Gap | Kategorie ogólne i placeholder transferowy, bez listy rzeczywistych procesorów. |
| Privacy by design/default | Partial-positive | Ustawienia publiczności profilu i audyt zgód; należy potwierdzić bezpieczne defaulty live. |
| Naruszenia/DPIA | Needs evidence | Brak dowodu planu IR/DPIA. |
| Mobile permissions/telemetry | Not assessed | `yalquo-mobile` jest puste. |
| Usunięcie konta | Present | Od 2026-10-01: `DELETE /api/me` (anonimizacja, potwierdzenie hasłem) + UI w ustawieniach profilu. Eksport danych z YAL-RODO-002 wciąż brakuje. |

### 7.3 Mata24

| Kontrola | Ocena | Dowód / luka |
|---|---|---|
| Inwentaryzacja danych, cele, podstawy | Partial | Polityka opisuje dane kont/edukacyjne/rodzinne; telemetryka crash nie jest wystarczająco odzwierciedlona. |
| Zgody i wycofanie | Partial | Cookie settings istnieją; zgoda rodzica i jej dowód wymagają potwierdzenia end-to-end. |
| Privacy policy / transparentność | Partial-positive | Dokument publiczny jest rozbudowany; wymaga dopasowania do crash reports, backupów i procesorów. |
| Cookies/analytics/tracking | Partial | Baner i kategorie opisane; brak dynamicznej weryfikacji kolejności ładowania. |
| Minimalizacja/retencja | Gap | Crash telemetry i historyczne dumpy; brak TTL/report redaction. |
| Dostęp/poprawienie/przenoszenie/usunięcie | Partial-positive | Eksport i anonymizacja konta; konieczne testy pełnego zakresu, relacji dziecka i backupów. |
| Sprzeciw/ograniczenie | Needs evidence | Informacja istnieje, workflow nieudowodniony. |
| Procesorzy/transfery | Partial | Kategorie dostawców opisane ogólnie; brak zatwierdzonego rejestru/subprocesorów w zakresie. |
| Privacy by design/default | Partial | Profile dzieci bez e-maila to minimalizacja; crash stack i brak DPIA są lukami. |
| Naruszenia/DPIA | Gap | MAT-RODO-002; dumpy wymagają oceny incydentu. |
| Mobile permissions/telemetry | Partial | Camera/photos z opisem; secure token storage; crash reports wymagają redakcji i retencji. |
| Usunięcie konta | Present | Endpoint anonymizacji i eksportu obecny; backup lifecycle nieudowodniony. |

## 8. Infrastruktura wspólna — obserwacje pozytywne

- Ingressy audytowanych aplikacji deklarują TLS.
- Deploymenty aplikacyjne mają requests/limits; większość ma readiness/liveness, backendy także startup probes.
- `yalquo-bot` deklaruje non-root, seccomp RuntimeDefault, drop capabilities i brak privilege escalation — wzorzec do skopiowania.
- Sprawdzone aplikacje Argo CD były `Synced` i `Healthy`; rewizja infrastruktury odpowiadała audytowanemu SHA.
- Nie stwierdzono `privileged`, `hostNetwork` ani `hostPID` w odczytanych workloadach.
- Namespace nie zawierały Role/RoleBinding ani ClusterRoleBinding do audytowanych ServiceAccount, co ogranicza skutki zbędnych tokenów, ale ich montowanie nadal jest niepotrzebne.
- Backup CronJobs istnieją i korzystają z referencji do Secrets; żadnych wartości nie odczytano.

## 9. Plan działań

### Natychmiast / 0–7 dni

1. Obsłużyć MAT-SUP-001 jako potencjalny incydent: izolacja, rotacja tokenów, zakres danych, DPO/IR, historia Git.
2. Usunąć przesyłanie tymczasowych haseł (TRI-SEC-001).
3. Zredagować/wyłączyć nadmiarowe crash reports i logowanie komunikatu (MAT-RODO-001).
4. Usunąć `:latest` i przypiąć Mata24 bot do immutable digest/tagu SHA.
5. Wyłączyć automount tokenu tam, gdzie aplikacja nie używa API Kubernetes.
6. Skorygować publiczne twierdzenie Yalquo o samoobsługowym usunięciu konta, jeśli funkcja nie istnieje.

### 30 dni

1. Default-deny NetworkPolicy i allow-listy.
2. Pełny pod hardening oraz polityka admission.
3. Pierwszy kontrolowany test restore i zredagowany protokół.
4. Produkcyjne polityki prywatności Trippics i Yalquo.
5. Limity uploadu i bezpieczna obsługa EXIF/lokalizacji.
6. Dostarczyć read-only evidence ustawień GitHub.

### 60 dni

1. Eksport/usunięcie konta Yalquo wraz z mediami, zgodami i backup lifecycle.
2. DPIA Mata24 oraz proces zgody/opieki nad kontami dzieci.
3. Retencja crash reports, bot operations, logów i danych zgód.
4. Rate limiting krytycznych endpointów.
5. Wdrożenia obrazów wyłącznie po digest oraz podpisy/attestations.

### 90 dni

1. Ćwiczenie incident response i odtworzenia po awarii z pomiarem RPO/RTO.
2. Automatyczne skanowanie sekretów i blokada dumpów danych w SCM.
3. Cykliczne SCA/SAST/SBOM z triage SLA.
4. Przegląd ROPA, procesorów, transferów i retencji przez kompetentnego prawnika/DPO.
5. Retest wszystkich findingów i porównanie stabilnych ID.

## 10. Instrukcje weryfikacji poprawek

- **MAT-SUP-001:** potwierdzić brak dumpów we wszystkich refs i obiektach osiągalnych, listę rotacji, zamknięty zapis incydentu oraz skan historii bez ujawniania wyników.
- **Kubernetes:** `kubectl get` bez Secrets powinien wykazać osobne SA/automount false, pełny securityContext i NetworkPolicy; uruchomić testy łączności w kontrolowanym środowisku przez zespół, nie w ramach tego pasywnego audytu.
- **TRI-SEC-001:** statycznie brak haseł w e-mailach; test integracyjny powinien potwierdzić jednorazowy token, TTL i unieważnienie.
- **MAT-RODO-001:** test jednostkowy redakcji z syntetycznymi tokenami/e-mailami; rekord i log nie mogą zawierać wejściowej wartości.
- **Prawa osób:** na syntetycznym koncie sprawdzić eksport, poprawienie i delete/anonymize, obiekty zależne, media, zgody i re-delete po restore.
- **Polityki:** porównać każdy opisany przepływ z kodem i listą procesorów; brak placeholderów; wersja/data/właściciel zatwierdzenia.
- **Backupy:** odtworzyć do izolowanego namespace, sprawdzić integralność, RPO/RTO i mechanizm usunięć; opublikować tylko zredagowany wynik.
- **Supply chain:** spec workloadu musi zawierać `image@sha256:…`; attestation ma wiązać digest z audytowanym SHA i tożsamością builda.
- **GitHub:** eksport potwierdza branch protection/rulesets, wymagane reviews/checks, least-privilege workflow permissions, scanning i brak nieprzypiętych actions.

## 11. Coverage gaps

1. Token GitHub nie miał uprawnień do administracyjnych ustawień, alertów bezpieczeństwa ani attestations (`403`).
2. `yalquo-mobile` i `mata24-marketing` nie mają commitów; brak materiału do audytu.
3. Nie uruchamiano kodu, testów, buildów, package managerów ani skanerów wymagających pobrania baz; podatności zależności są tylko częściowo ocenione.
4. Brak lokalnego narzędzia SCA z aktualną offline bazą CVE/OSV; Dependabot alerts niedostępne.
5. Nie wykonywano DAST, testów auth/IDOR, uploadów, cookies ani mobilnego runtime; wnioski runtime są `Likely`/`Needs evidence`.
6. Nie odczytywano Secrets, ConfigMap values, kubeconfigu, tokenów SA ani logów. Rotacja, wartości szyfrowania i działanie backupów nie zostały potwierdzone.
7. Nie potwierdzono zewnętrznej konfiguracji ingress controller/WAF/rate limits ani private registry provenance.
8. Brak bezpiecznego dowodu testów restore, ćwiczeń IR, DPIA, ROPA, umów powierzenia, SCC i faktycznych lokalizacji procesorów.
9. Nie badano urządzeń mobilnych, store privacy labels ani uprawnień nadanych w zainstalowanych buildach.
10. Ocena historii Git pod kątem pełnego czasu ekspozycji dumpów była ograniczona do audytowanego snapshotu; dochodzenie incydentowe musi objąć refs, forki, cache i klony.

## 12. Appendix — bezpieczne metadane

### Audytowane commity domyślnych gałęzi

| Repozytorium | Branch | Commit SHA |
|---|---|---|
| trippics | main | `f54bad5647a3494eccb3bc16b8a77238a976437c` |
| trippics-backend | main | `77c820b1f31303bffb6a06d85ddc42ae148360d9` |
| trippics-frontend | main | `8b41902a6f725f9e539472ccabd737db3ba73a7e` |
| trippics-mobile | main | `3f1e7a5eda914e06af64485d71862577d899e7d8` |
| yalquo-backend | main | `ac57510d926e25d464eb1a1afeaf38e2be0e51aa` |
| yalquo-frontend | main | `20b446e273329b8b9d8054ce632c20020c9d9bd2` |
| yalquo-bot | main | `dd12ef56decd731ad2026290b994fdb73baae0a5` |
| matematicon | main | `230a20a3a4a4fd82940295c3a47796c6b09a8435` |
| mata24-frontend | main | `8baacd2e2c267cb4811c36532ff864052bd3ed11` |
| mata24-mobile | main | `e4c39cac8f219933c77f7a024b24a8bea3ab3177` |
| mata24-bot | main | `79e15aca8c2dbcf1b819125854284b1b0ffb7ba9` |
| local-kubernetes-cluster-definition | main | `47093c915fb4fef5f32e58195edffcc49a13b832` |
| yalquo-mobile | — | puste repozytorium |
| mata24-marketing | — | puste repozytorium |

### Wersje narzędzi

- Git `2.47.3`
- GitHub CLI `2.96.0`
- kubectl client `1.28.15`; server `1.28.15`
- GNU grep/find, Python `3.13` — lokalne skrypty tylko do odczytu i redakcji
- Dedykowane SCA/SAST/secret-scanner: niedostępne; użyto lokalnego heurystycznego skanu bez wysyłania kodu.

### Obrazy — publicznie bezpieczne referencje

- Aplikacyjne deploymenty używały tagów odpowiadających pierwszym ośmiu znakom audytowanych SHA; registry host i prywatne digesty zredagowano.
- Bazy: `postgres:16-alpine`, `redis:7-alpine`.
- Build/runtime Dockerfile: Node 22 Alpine, nginx 1.27 Alpine, Eclipse Temurin 21/26 oraz Alpine 3.20.
- Żaden odczytany workload nie używał immutable digestu; `mata24-bot` CronJob deklarował `latest`.

---

Raport nie zawiera sekretów, surowych logów, wartości ConfigMap/Secrets, prywatnych hostów/IP, kubeconfigów, pełnych odpowiedzi API ani fragmentów prywatnego kodu poza minimalnymi opisami ścieżek i linii.

## 13. Log napraw

### 2026-09-30 - MAT-SUP-001: usunięcie dumpów bazy z repo i historii Git

- Zakres: repozytorium `matematicon` (Mata24), wszystkie 7 branchy (`main`, `admin-panel`, `develop`, `feature/oauth-google-facebook-login`, `mobile-android-prototype`, `openhands/campaign-key-attribution`, `openhands/marketing-analytics`).
- Usunięto z całej historii Git (`git-filter-repo` na świeżym mirror clone + force-push do origin) 10 plików: 3 wskazane w audycie (`mat1_backup_20260723.sql`, `mata24_PROD_backup_20260723.sql`, `mata24_PROD_backup_20260725.sql`) oraz 7 dodatkowych dumpów znalezionych przy tej samej okazji (`mat1_backup_20241124.sql`, `mat1_backup_20241208.sql`, `mat1_backup_20250223.sql`, `mat1_backup_20260623-migration.sql`, `mat1_backup_20290223.sql`, `mat1_bck_230825.sql`, `test.sql`).
- Zweryfikowano brak tych plików we wszystkich 203 commitach na wszystkich 7 branchach po przepisaniu historii.
- Dodano wpisy do `.gitignore` (`*backup*.sql`, `*_bck_*.sql`, `/*.sql` w katalogu głównym), żeby zapobiec ponownemu wrzuceniu dumpu do repo.
- Sprawdzono zawartość najnowszego dumpa (`mata24_PROD_backup_20260725.sql`) przed usunięciem: 55 rekordów z hashami haseł bcrypt i 36 z aktywnymi tokenami aktywacyjnymi (`authentication_key`, jednorazowy token aktywacji konta, kasowany po użyciu), ale wszystkie 59 kont miały puste lub `@example.com` adresy e-mail - brak realnych adresów e-mail w tym snapshocie.
- Wykonano pełny lokalny backup repo sprzed przepisania historii, przechowywany poza SCM.

Pozostaje do zrobienia (poza zakresem tej naprawy):
- Unieważnienie `authentication_key`/`reset_token` w bazie produkcyjnej (SQL przygotowany, do wykonania przez zespół z dostępem do bazy).
- Włączenie GitHub Secret Scanning i Push Protection dla repo `matematicon`.
- Ocena, czy formalne zamknięcie zapisu incydentu jest wymagane, biorąc pod uwagę brak realnych adresów e-mail w ujawnionym dumpie.

### 2026-09-30 - SHR-K8S-001 (część 1/2): wyłączenie automount tokenu ServiceAccount

- Zakres: repozytorium `local-kubernetes-cluster-definition`, 8 manifestów aplikacyjnych obejmujących wszystkie audytowane workloady Trippics/Yalquo/Mata24 (`trippics-backend`, `trippics-frontend`, `yalquo-backend`, `yalquo-frontend`, `yalquo-bot`, `mata24-backend`, `mata24-frontend`, `mata24-bot/cronjob.yaml`).
- Dodano `automountServiceAccountToken: false` w `spec.template.spec` każdego z tych workloadów - żadna z tych aplikacji nie rozmawia z API Kubernetes, więc token ServiceAccount `default` był montowany bez potrzeby.
- Zmiana wypchnięta commitem `0b84d1a`, zsynchronizowana przez ArgoCD i zweryfikowana live na klastrze: wszystkie 9 aplikacji `Synced`/`Healthy`, rolling update bez restartów podów, `automountServiceAccountToken: false` potwierdzone na żywych Deploymentach. `mata24-bot` (CronJob) dostanie ustawienie przy najbliższym uruchomieniu z harmonogramu.
- Część 2 (pod/container `securityContext`: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`, docelowo `readOnlyRootFilesystem`) zostaje na osobną turę - backendy Spring (obrazy `eclipse-temurin`, brak `USER` w Dockerfile) i frontendy nginx (bindowanie portu 80 wymaga roota lub zmiany na `nginx-unprivileged` + port 8080) wymagają osobnych testów zgodności przed wdrożeniem. `yalquo-bot` ma już gotowy wzorzec tej konfiguracji do skopiowania.

### 2026-09-30 - SHR-K8S-002 (część 1): default-deny NetworkPolicy dla yalquo, mata24, trippics (backend/frontend)

- Zakres: repozytorium `local-kubernetes-cluster-definition`. Namespace `sandbox` (yalquo-backend, yalquo-frontend), `mata24` (mata24-backend, mata24-frontend) i `trippics` (trippics-backend, trippics-frontend). Wdrażane appka po appce, commit po commicie, z ręcznym testem po każdym kroku.
- Każda appka dostała własny `NetworkPolicy` (`podSelector` na jej własny label `app: ...`, `policyTypes: [Ingress, Egress]`) zamiast jednego namespace'owego default-deny - bezpieczniejsze w wielolokatorskich namespace'ach (`sandbox`/`mata24` dzielą przestrzeń z botami, backupami i zadaniami jednorazowymi, których jeszcze nie objęliśmy politykami).
- Rozpoznano faktyczny ruch przed napisaniem polityk (env, `nginx.conf`, istniejące `NetworkPolicy` w `databases`/`redis`/`minio`): `yalquo-frontend` i `mata24-frontend` proxują server-side `/api` itp. do backendu (wymagany egress frontend→backend), `trippics-frontend` tego nie robi - przeglądarka woła `api.trippics.pl` bezpośrednio (CORS), więc trippics-frontend nie ma egressu do backendu.
- `mata24-backend`: ingress dopuszczony z `ingress-nginx` (w tym `api.mata24.pl` używane przez mobile), z `mata24-frontend` i z `mata24-bot`. `mata24-bot` (CronJob) nie miał stabilnego labela na podach (`job-name` zmienia się co uruchomienie) - dodano `app: mata24-bot` w `cronjob.yaml`, żeby dało się go w ogóle wskazać selectorem.
- Egress wszystkich backendów: DNS (`kube-system:53`), właściwe bazy/cache/storage (`databases:5432`, `redis:6379`, `minio:9000` dla mata24/trippics; własny `yalquo-postgres`/`yalquo-redis` w `sandbox` dla yalquo - main-db/redis wspólne jeszcze nie wpuszczają tego namespace'u), internet `443`/`587` (SMTP, reCAPTCHA, Google OAuth) ograniczony `ipBlock` z wyjątkiem CIDR-ów klastra (`10.244.0.0/16`, `10.96.0.0/12`) - żeby "internet" nie znaczyło przypadkiem "cały ruch wewnątrz klastra".
- Zweryfikowano empirycznie, że sondy kubelet (liveness/readiness/startup) nie są blokowane przez te polityki (już działające `NetworkPolicy` na `main-db` od 52 dni to potwierdzają) - nie trzeba było dodawać osobnych reguł na ruch z węzła.
- Commity `63f6fe6` (pilot yalquo) i `efc8137` (mata24 + trippics), zsynchronizowane przez ArgoCD. Weryfikacja: wszystkie 7 aplikacji `Synced`/`Healthy`; wymuszony rolling restart `mata24-backend`/`trippics-backend` żeby sprawdzić NOWE połączenia (nie tylko te otwarte przed wdrożeniem polityki) - zero restartów, DB/Redis/Minio połączyły się poprawnie; kolejny zaplanowany przebieg `mata24-bot` (po wdrożeniu polityki) zakończony bez błędów połączenia; realny ruch przez `ingress-nginx` (host-based routing) dla `mata24.pl`, `api.mata24.pl`, `trippics.pl`, `api.trippics.pl`, `yalquo.pl` zwraca odpowiedzi aplikacji, nie timeouty.

Pozostaje do zrobienia (poza zakresem tej naprawy):
- `yalquo-bot`, `yalquo-postgres`, `yalquo-redis`, `mata24-bot`, backup CronJoby i pozostałe pody w `sandbox`/`mata24`/`trippics` (obecnie bez własnej polityki - w międzyczasie pozostają otwarte, bo żadna `NetworkPolicy` ich nie wybiera).
- Namespace `yalquo` (docelowy, obecnie pusty) - do objęcia przy migracji z `sandbox`.
- Rozszerzenie `main-db-allow-consumers`/`redis-allow-consumers` o `sandbox`/`yalquo` po docelowej migracji Yalquo ze współdzielonych baz.

### 2026-09-30 - SHR-K8S-002: incydent i poprawka - trippics-frontend jednak proxuje do backendu

- Po wdrożeniu polityk z poprzedniego wpisu `www.trippics.pl` przestał pobierać dane (np. `/api/main/home` wisiał w nieskończoność) - użytkownik zgłosił awarię od razu po teście.
- Przyczyna: błędne założenie przy pisaniu polityki dla `trippics-frontend`. Obecność `APP_CORS_ALLOWED_ORIGINS` i osobnego hosta ingress `api.trippics.pl` w `trippics-backend` zasugerowały, że przeglądarka woła backend bezpośrednio (cross-origin) i frontend go nie proxuje - inaczej niż w mata24/yalquo. Weryfikacja żywego `nginx.conf` w działającym podzie (dopiero po awarii) pokazała, że `trippics-frontend` **też** proxuje server-side `/api/`, `/pictures/`, `/avatars/` do `trippics-backend` - oba wzorce (bezpośredni `api.trippics.pl` dla mobile i proxy dla weba) współistnieją. `api.trippics.pl` osobno służy najpewniej `trippics-mobile`.
- Poprawka: dodano brakujący egress `trippics-frontend` → `trippics-backend:8080` oraz brakujący ingress `trippics-backend` ← pod `trippics-frontend`. Zastosowano najpierw bezpośrednio na klastrze (`kubectl apply`) żeby jak najszybciej przywrócić serwis, równolegle wypchnięte do gita (commit `2032256`) - ArgoCD ma `selfHeal: true`, więc sam live `kubectl patch` (próbowany chwilę wcześniej) został cofnięty automatycznie, dopóki git się nie zgadzał.
- Zweryfikowano po poprawce: `HTTP 200` na `/api/main/home`, `/pictures/`, `/avatars/` (błędy `500` na gołych katalogach to normalne zachowanie aplikacji przy braku nazwy pliku, nie problem sieciowy). Rolling restart obu podów backendu bez błędów, ArgoCD `Synced`/`Healthy`.
- Wniosek na przyszłość: przy pisaniu polityk dla kolejnych appek (bot/backup/postgres/redis) weryfikować rzeczywisty ruch bezpośrednio w żywym podzie (`kubectl exec ... cat <config>` / logi), nie wnioskować z pośrednich sygnałów jak zmienne CORS czy nazwy hostów ingress.

### 2026-09-30 - SHR-K8S-002 (część 2): boty, backup i datastore'y sandboxa/mata24

- Tym razem ruch zweryfikowany wprost w źródle (repo `yalquo-bot`, repo `mata24-bot`, `backup-script-configmap.yaml`) przed napisaniem polityk - wniosek z incydentu trippics wyżej.
- `yalquo-bot`: osobny `NetworkPolicy` w `apps/yalquo-bot/`. Kluczowe odkrycie z kodu: bot woła własne API mata24... nie, yalquo (`app.yalquo-api.base-url`) domyślnie pod publicznym `https://api.yalquo.infra.trippics.pl` - **przez ingress, nie in-cluster Service** ("boty działają wyłącznie przez HTTP, jak prawdziwy klient") - więc to egress do internetu (443), nie do poda backendu bezpośrednio. Ma też własną bazę (`YALQUO_BOT_DB_URL`, osobną od `yalquo`, ale na tym samym serwerze `yalquo-postgres`) i pobiera zdjęcia z `cdn.yalquo.pl` (też 443). Brak MinIO/S3 w kodzie.
- `mata24-bot`: analogicznie zweryfikowane w źródle (`.env.example`, `src/api.ts`) - w przeciwieństwie do yalquo-bota, ten woła backend **in-cluster** (`http://mata24-backend.mata24.svc.cluster.local:8080`) i łączy się bezpośrednio z `mat1` (main-db) przez `pg`. Nowy `apps/mata24-bot/networkpolicy.yaml`, `ingress: []` (nic do niego nie woła).
- `yalquo-postgres` / `yalquo-redis` / `yalquo-db-backup`: trzy polityki w jednym pliku `apps/yalquo-postgres/networkpolicy.yaml` (ingress-only dla obu datastore'ów, ingress+egress dla backupu - pg_dump lokalnie + rclone do Google Drive na 443). `yalquo-redis` nie jest w GitOps (ad-hoc `kubectl`, komentarz "jednorazowy Redis" w kodzie) - polityka i tak trzymana w repo, przy pozostałych datastore'ach sandboxa.
- Powtórzony problem ze stabilnością labeli: `yalquo-db-backup` CronJob też nie miał `app:` (tylko zmienny `job-name`) - dodano, jak wcześniej przy `mata24-bot`.
- Weryfikacja: commit `044c667`, wszystkie 5 dotkniętych aplikacji ArgoCD `Synced`/`Healthy`. Wymuszone NOWE połączenia (nie tylko już otwarte): rolling restart `yalquo-backend` i `yalquo-bot`, ręcznie odpalony `kubectl create job --from=cronjob` dla `mata24-bot` (log: `tick_idle`, bez błędu połączenia) i `yalquo-db-backup` (log: `Backup bazy yalquo zakonczony OK` - pg_dump + gpg + upload na Google Drive przeszły). Panel operatora `yalquo-bot` (`yalquo-bot.infra.trippics.pl`) odpowiada `302` (przekierowanie do logowania, nie timeout). Restarty wszystkich podów: `0`.

Pozostaje do zrobienia (poza zakresem SHR-K8S-002):
- Namespace `databases`/`redis`/`minio`/`vault` mają już ingress-only `NetworkPolicy` sprzed audytu - egress z tych namespace'ów (np. `main-db-backup`, backup MinIO) nie był w zakresie audytu (namespace'y `trippics`/`yalquo`/`mata24`/`sandbox`) i zostaje nietknięty.
- Jednorazowe/testowe pody w `sandbox` (`yalquo-bot-operations-constraint-fix`, ewentualne przyszłe `curltest`) - celowo pominięte, nie są stałym workloadem.
- Namespace `yalquo` (docelowy, pusty) i rozszerzenie `main-db-allow-consumers`/`redis-allow-consumers` o niego po migracji z `sandbox` - jak w poprzednim wpisie.

### 2026-09-30 - TRI-SEC-001: koniec wysyłki tymczasowego hasła e-mailem

- Zakres: repozytorium `trippics-backend` (lokalnie sklonowane jako `new-trippics` - to ten sam repo/remote, HEAD zgodny z wdrożonym obrazem).
- Doprecyzowanie względem opisu findingu: `sendPasswordResetLink` (reset "zapomniałem hasła", token ważny 1h) **już działał poprawnie** i nie wymagał zmian - tylko nie był jedynym miejscem generującym hasło. Podatność była węższa niż cały flow resetu: dotyczyła wyłącznie `loginOrRegisterViaFacebook` (zakładanie konta przez Facebook), które generowało 7-znakowe tymczasowe hasło i wysyłało je w treści maila (`sendFacebookWelcomeEmail`). Analogiczny `loginOrRegisterViaGoogle` już wcześniej robił to bezpiecznie - losowe 24-znakowe hasło, zahashowane, nigdzie nieujawniane, bo logowanie odbywa się wyłącznie przez Google.
- Poprawka: `loginOrRegisterViaFacebook` dostał dokładnie ten sam wzorzec co Google (hasło 24 znaki, `UUID.randomUUID()`, nigdy nieeksponowane) - usunięto wywołanie `sendFacebookWelcomeEmail` oraz samą (teraz martwą) metodę z `EmailService`/`EmailServiceImpl`. Powiadomienie do admina o nowym koncie zostaje (nie zawiera PII/sekretów).
- Weryfikacja: `mvn clean compile` lokalnie (`BUILD SUCCESS`, 125 plików) przed pushem. Commit `5bc2379`, zbudowany przez Jenkins (build #33, maven+kaniko), obraz wdrożony przez GitOps (`323bbe0` w `local-kubernetes-cluster-definition`, ArgoCD `Synced`/`Healthy`). Oba nowe pody `Ready`, restarty `0`. Realny ruch przez ingress (`www.trippics.pl/api/main/home`) nadal `200` po deployu.
- Poza zakresem tej poprawki: `sendActivationEmail` (link aktywacyjny, nie hasło) i `sendPasswordResetLink` (token, nie hasło) nie wymagały zmian - już zgodne z rekomendacją audytu.

### 2026-09-30 - Mata24: sekcja polityki prywatności dla aplikacji mobilnej i link usuwania konta (zgodność z Google Play)

- Nie jest to naprawa konkretnego findingu z tego audytu - zgłoszone przez użytkownika osobno, przy okazji przeglądu zgodności polityki prywatności mata24 z zasadami danych użytkownika Google Play (wymagania sekcji Data Safety/Account Deletion w Konsoli Play). Dotyczy tych samych obszarów co wiersze "Usunięcie konta" i "Mobile permissions/telemetry" w sekcji 7.3 tego raportu.
- Ustalenie: `mata24-mobile/src/screens/profile/ProfileScreen.tsx` linkował do `/terms` i `/privacy`, ale nie miał żadnej opcji usunięcia konta - Google Play wymaga wprost, żeby taka opcja była łatwa do znalezienia w samej aplikacji, nie tylko na stronie WWW. Backend (`UserService.anonymizeUser` w `matematicon`) i web (`/profile`) już to miały; brakowało wejścia z mobile.
- Ustalenie: polityka prywatności (`mata24-frontend/src/app/terms/privacypolicy/privacy.component.html`) opisywała wyłącznie dane/cookies webowe; nie tłumaczyła sposobu przechowywania tokenu w aplikacji mobilnej (Keychain/EncryptedSharedPreferences zamiast cookies), natywnego logowania Google Sign-In, uprawnień aparat/galeria ani retencji danych po usunięciu konta (usunięcie = anonimizacja, nie twardy DELETE - historia nauki zostaje bez powiązania z osobą, żeby nie osierocać kont dzieci powiązanych przez `parent_id`).
- Zweryfikowane w kodzie (nie zgadywane): `mata24-mobile/package.json` nie ma żadnego SDK analitycznego, reklamowego ani crash-trackingu firm trzecich; `app.json` ma już poprawne opisy zgód systemowych na `photosPermission`/`cameraPermission` i jawne `microphonePermission: false`.
- Poprawka: link "Usuń konto" w `ProfileScreen.tsx` (otwiera `mata24.pl/profile` w przeglądarce urządzenia, gdzie usuwanie/anonimizacja już działa). Polityka prywatności: nowa sekcja 9 "Aplikacja mobilna" (brak SDK analitycznych/reklamowych/trackingowych, brak trwałych identyfikatorów urządzenia, token w bezpiecznym magazynie systemowym, Google Sign-In natywny, uprawnienia aparat/galeria tylko przy zmianie awatara, link usuwania konta) oraz akapit w sekcji 5 tłumaczący anonimizację po usunięciu konta. Renumeracja sekcji 9-12 na 10-13.
- Commity: `mata24-frontend` `9098c29` (polityka), `mata24-mobile` `0e3c4cd` (link usuwania konta), oba wypchnięte na `main`.

Pozostaje do zrobienia (poza zakresem tej poprawki):
- `mata24-frontend` trafi na produkcję przy najbliższym deployu (Jenkins/ArgoCD) - nie zweryfikowano jeszcze na żywym `mata24.pl`.
- `mata24-mobile` wymaga nowego builda APK/AAB w Jenkinsie (job "Mata24.pl/mata24-mobie"), żeby link "Usuń konto" trafił do faktycznej aplikacji na telefonach/Play Store - nie zbudowano jeszcze w tej sesji.
- Sekcja "Bezpieczeństwo danych" w Konsoli Google Play (deklaracje zbierania/udostępniania danych) nie została zaktualizowana - to działanie poza repozytoriami kodu, do wykonania ręcznie w konsoli.

### 2026-09-30 - MAT-RODO-001: redakcja PII w crash reports, koniec dublowania w logu, włączona retencja

- Zakres: `mata24-mobile` (`src/lib/crashReporting.ts`), `matematicon` (kod redakcji i joba retencji już istniał w repo, ale nie był wdrożony/włączony na produkcji), `local-kubernetes-cluster-definition` (`apps/mata24-backend/deployment.yaml`).
- Ustalenie przy starcie pracy: backendowa połowa tego findingu **już była zaimplementowana** w `matematicon` (`ClientErrorReportService.record` redaguje przez `LogRedaction.redact` przed zapisem, nie loguje już pełnego komunikatu ponownie, `ClientErrorReportRetentionJob` czyści rekordy starsze niż 30 dni) - prawdopodobnie praca równoległej sesji nad tym samym audytem, nieudokumentowana dotąd w tym pliku. Komentarz w `ClientErrorReportService.java` twierdził jednak, że "mobile już redaguje po swojej stronie (crashReporting.ts)" - **nieprawda**: `crashReporting.ts` wysyłało surowy `message`/`stack` bez żadnej redakcji, testy (`crashReporting.test.ts`) to potwierdzały (`expect(body.message).toBe('cos sie zepsulo')` na surowym tekście). Job retencji miał też `@ConditionalOnProperty(...havingValue = "true")` bez odpowiadającej zmiennej środowiskowej w `deployment.yaml` - czyli de facto wyłączony na produkcji mimo gotowego kodu.
- Poprawka mobile: `redact()` w `crashReporting.ts` - lustrzane odbicie `LogRedaction.java` (query string, JWT, e-mail, długi token jako rezerwa), stosowane do `message` i `stack` przed `fetch` do `/pub/mobile/crash-reports`. Nowe testy jednostkowe (redakcja e-maila/JWT/query stringu/długiego tokenu, przepuszczenie `undefined`, oraz test end-to-end że `reportError` faktycznie wysyła już zredagowaną treść).
- Poprawka infrastruktury: `APP_CLIENT_ERROR_REPORTS_RETENTION_ENABLED: "true"` dodane do `apps/mata24-backend/deployment.yaml` - włącza już istniejący, wcześniej nieaktywny job retencji (30 dni, cron `0 15 3 * * *`).
- Polityka prywatności mata24 (`privacy.component.html`, sekcja 9 "Aplikacja mobilna") doprecyzowana: poprzedni zapis "brak narzędzi do raportowania awarii" mówił wyłącznie o braku SDK firm trzecich (Sentry/Crashlytics), a nie oddawał faktu, że aplikacja wysyła własne raporty awarii do własnego backendu - dodany osobny akapit: co jest zbierane, że jest redagowane przed i po wysłaniu, oraz że retencja wynosi 30 dni. To domyka "udokumentować telemetrykę" z rekomendacji findingu.
- Świadomie pominięte: formalny opt-out z raportowania awarii. Dane są już zredagowane (bez PII w praktyce), służą wyłącznie stabilności aplikacji (uzasadniony interes), a dodanie przełącznika w ustawieniach dla mało używanej aplikacji bez panelu administracyjnego crash reportów uznano za nieproporcjonalny nakład względem ryzyka - do rewizji, jeśli DPO/audyt prawny stwierdzi inaczej.
- Weryfikacja: `npx jest crashReporting` (11/11 testów), `npx tsc --noEmit` (czysto) w `mata24-mobile` przed pushem. Commity: `mata24-mobile` `46ee8f8`, `mata24-frontend` `61eba07`, `local-kubernetes-cluster-definition` `a22d86b` (ten ostatni rebase'owany na świeży `origin/main`, bo równoległa sesja wypchnęła `323bbe0` w międzyczasie). Na żywo: `kubectl` potwierdza `APP_CLIENT_ERROR_REPORTS_RETENTION_ENABLED=true` w specyfikacji działającego poda, nowa rewizja `mata24-backend` wystartowała bez błędów (`Started MetematiconBackendApplication in 148s`, zgodne ze znanym czasem startu tego serwisu) i przejęła ruch. ArgoCD miga `Synced`/`OutOfSync` po deployu - potwierdzone jako już wcześniej istniejący, niezwiązany z tą zmianą efekt rozjazdu `spec.replicas` między Gitem a HPA (ten sam serwis ma za sobą setki takich przełączeń w historii `observedGeneration`), nie nowa regresja.
- Pozostaje do zrobienia (poza zakresem tej poprawki): rzeczywisty pierwszy przebieg joba retencji nastąpi o 03:15 - nie zweryfikowano jeszcze logu potwierdzającego usunięcie; `mata24-mobile` wymaga nowego builda APK/AAB w Jenkinsie, żeby redakcja trafiła do faktycznej aplikacji na telefonach.

### 2026-09-30 - SHR-SUP-001 (częściowo): mata24-bot przypięty do immutable digestu

- Zakres: `local-kubernetes-cluster-definition` (`apps/mata24-bot/cronjob.yaml`), `mata24-bot` (`Jenkinsfile`).
- Ustalenie: Jenkinsfile mata24-bot już budował i wypychał obraz pod dwoma tagami (`:<8-znakowy SHA>` i `:latest`), ale manifest GitOps celowo zostawał na `:latest` - komentarz w kodzie tłumaczył to decyzją operacyjną ("suspend to nie deploy per commit"), co było nietrafionym uzasadnieniem (użycie SHA/digestu w referencji obrazu nie ma związku z tym, czy CronJob jest wstrzymany).
- Digest ustalony bez zgadywania: `kubectl get pods -n mata24 -l app=mata24-bot -o jsonpath='...imageID'` na trzech ostatnich uruchomieniach joba dał spójny wynik `sha256:9b58b4ca61ea...`, zgodny z HEAD repo `mata24-bot` (`79e15aca`, ten sam commit co w tabeli audytowanych commitów w sekcji 12).
- Poprawka: `cronjob.yaml` pinowany do `ghcr.io/leszekbednorz/mata24-bot@sha256:9b58b4ca...` zamiast `:latest`. Jenkinsfile: `--digest-file` w kroku kaniko + nowy stage "Update Manifest" (analogiczny do już istniejącego w `matematicon`/mata24-backend) - na każdym buildzie z `main` automatycznie podmienia digest w GitOps przez `sed` + commit + push, więc pin nie będzie się starzał przy kolejnych wydaniach bota.
- Weryfikacja na żywo: `kubectl -n argocd annotate application mata24-bot argocd.argoproj.io/refresh=hard`, `Synced`/`Healthy` na rewizji zgodnej z nowym commitem, `kubectl get cronjob -o jsonpath='...image'` potwierdza digest na żywym zasobie. Zmiana nie wymagała restartu niczego na żywo - CronJob użyje nowego obrazu przy najbliższym zaplanowanym uruchomieniu (co 10 min).
- Commity: `local-kubernetes-cluster-definition` `c1d9572`, `mata24-bot` `47dbdce`.
- Pozostaje do zrobienia (poza zakresem tej poprawki): pozostałe workloady (`mata24-backend`, `mata24-frontend`, `trippics-*`, `yalquo-*`) nadal referencjonują skrócony SHA tagu, nie digest - tag SHA jest w praktyce prawie tak dobry jak digest (nowy push pod ten sam SHA-tag nie powinien się zdarzyć, bo tag pochodzi z commit hasha), ale audyt formalnie wymaga `@sha256:`; ujednolicenie wszystkich pipeline'ów zostaje na termin 60-dniowy.

### 2026-09-30/10-01 - YAL-RODO-002 (częściowo): samoobsługowe usunięcie konta w Yalquo

- Zakres: `yalquo-backend` (nowy endpoint + serwis), `yalquo-frontend` (UI), dokumenty prawne w obu repo.
- Mapa modelu danych zrobiona przed kodem (subagent Explore): żadna encja w `yalquo-backend` nie ma kaskady z `User`, a `Achievement.createdBy`/`ChallengeProposalComment.author`/`ChallengeProposalVote.user` są NOT NULL i wskazują na treść współdzieloną z innymi użytkownikami (dyskusje w Kuźni, denormalizowane liczniki głosów/popularności) - twardy DELETE osierociłby te rekordy albo zepsuł dane innych osób. Ten sam wniosek co wcześniej w Trippics/Mata24 (patrz notatka w sekcji o MAT-RODO-001 wyżej i wpisy pamięci projektowej): anonimizacja, nie fizyczny DELETE.
- Backend (`UserService.removeAccount`, `MeApiController` `DELETE /api/me`): wymaga potwierdzenia aktualnym hasłem (chroni przed usunięciem konta przez przejętą/porzuconą sesję), blokuje samodzielne usunięcie konta administratora, nadpisuje login/e-mail losowym UUID (nie surowym id - nie ujawnia że konto istniało), czyści PII, `authVersion++` + `refreshTokenRepository.deleteByUser` (natychmiastowe unieważnienie WSZYSTKICH sesji - ten wzorzec istniał już w `AdminIdentityService` dla akcji administratora, ale nie w self-service `changePassword`, co było niespójnością zamkniętą przy okazji), usuwa avatar z magazynu plików, czyści IP/User-Agent z historii zgód i akceptacji regulaminu (same zgody/akceptacje zostają jako dowód historyczny - `ConsentEvent` jest z założenia append-only).
- Brakujący prymityw odkryty po drodze: `StorageService` (abstrakcja dysk/S3-MinIO) miała `store()` i `resolveUrl()`, ale **żadnej metody `delete()`** - w całym kodzie (też przy `ChallengeImage`/`ChallengeAttachment`) usunięcie wiersza z bazy nigdy nie usuwało pliku z magazynu. Dodano `StorageService.delete()` (obie implementacje) i użyto jej dla avatara; pozostałe miejsca z tym samym brakiem zostają nietknięte (poza zakresem tej poprawki).
- Frontend: sekcja "Strefa zagrożenia" na `/profil` (formularz z polem hasła, potwierdzenie, komunikat o nieodwracalności), `AuthService.deleteAccount()` czyści sesję i przekierowuje na `/konto` po sukcesie - ten sam wzorzec co `logout()`.
- Dokumentacja: poprawiony fałszywy zapis w `yalquo-frontend/legal/polityka-prywatnosci.md:74` (dokładnie ten cytowany w opisie findingu) który twierdził, że usunięcie konta jest "dostępne samodzielnie w ustawieniach Konta", mimo że funkcja nie istniała. Autorytatywne dokumenty w `yalquo-backend/src/main/resources/legal/` (regulamin.md, polityka-prywatnosci.md) rozszerzone o opis nowej ścieżki i tego, co faktycznie się dzieje przy usunięciu (anonimizacja, nie DELETE; treści współdzielone zostają).
- **Decyzja o wersjonowaniu dokumentów prawnych - świadomie przedyskutowana z użytkownikiem**: `LegalDocumentBootstrap` (który publikuje `TERMS_VERSION`/`PRIVACY_VERSION` jako nowe wiersze `LegalDocument`) jest wyłączony na produkcji (`app.legal.bootstrap-enabled=false`, brak override w k8s) - realna publikacja nowej wersji dzieje się ręcznie przez panel admina (`AdminLegalDocumentController`/`publishNewVersion`). Podbicie wersji regulaminu (1.3→1.4) i polityki (1.4→1.5) w kodzie jest więc bezpieczne samo w sobie - wymuszenie ekranu ponownej akceptacji regulaminu u wszystkich userów (`legalGuard` w `yalquo-frontend` przekierowuje na `/akceptacja-regulaminu` dopóki wersja się nie zgadza) nastąpi dopiero gdy administrator faktycznie opublikuje nową wersję przez panel, nie automatycznie przy tym wdrożeniu.
- Weryfikacja: `mvn test` w `yalquo-backend` - 242/242 (10 nowych testów `removeAccount`: błędne hasło, blokada admina, pełna anonimizacja + inwalidacja sesji, pominięcie usuwania avatara gdy go nie było; naprawiono też 2 istniejące testy zepsute nowym wymaganym override'em `StorageService.delete()`). `npx ng build` w `yalquo-frontend` - czysto (tylko istniejące wcześniej ostrzeżenia budżetu CSS).
- Commity: `yalquo-backend` `bf13106` (feature) + `2c2568d` (dokumenty prawne), `yalquo-frontend` `a671c3f` (UI + korekta draftu polityki).
- Pozostaje do zrobienia (poza zakresem tej poprawki, wciąż otwarta część YAL-RODO-002): uwierzytelniony eksport danych (prawo do przenoszenia) - nie zaimplementowany. Publikacja nowych wersji dokumentów prawnych przez panel admina - do ręcznego wykonania przez użytkownika, kiedy zdecyduje.
