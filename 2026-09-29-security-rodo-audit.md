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

### SHR-K8S-001 — niepełny hardening podów i automatyczne tokeny ServiceAccount

- **Aplikacja i komponent:** wspólne; deploymenty Trippics, Yalquo i Mata24.
- **Kategoria:** Kubernetes hardening.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** live deploymenty używają `serviceAccountName: default`, nie ustawiają `automountServiceAccountToken: false`; poza `yalquo-bot` brak jawnych `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]` i `seccompProfile`. Repo: `apps/*/deployment.yaml`; live spec odczyt 2026-09-29.
- **Wpływ:** przejęty kontener otrzymuje zbędny token API i szersze możliwości eskalacji w obrębie poda/noda.
- **Rozwiązanie:** oddzielny minimalny ServiceAccount albo `automountServiceAccountToken: false`; pod/container security context z non-root, RuntimeDefault, drop ALL, no privilege escalation i read-only root filesystem po testach kompatybilności; egzekwować Pod Security Admission `restricted`.
- **Właściciel:** Platform/Kubernetes + zespoły aplikacyjne.
- **Termin:** 7 dni dla tokenów, 30 dni dla pełnego hardeningu.

### SHR-K8S-002 — brak NetworkPolicy

- **Aplikacja i komponent:** namespace `trippics`, `yalquo`, `mata24`, `sandbox`.
- **Kategoria:** segmentacja sieciowa.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** live API zwróciło zero `NetworkPolicy` we wszystkich czterech namespace; repo zawiera polityki dla części usług wspólnych, lecz nie dla audytowanych aplikacji.
- **Wpływ:** po kompromitacji jednego poda możliwy jest nieograniczony ruch lateralny zgodnie z możliwościami CNI i usług klastra.
- **Rozwiązanie:** default-deny ingress/egress, następnie allow-list DNS, ingress controller, wymagane bazy/cache/object storage i jawnie uzasadnione wyjścia internetowe.
- **Właściciel:** Platform/Kubernetes.
- **Termin:** projekt ≤7 dni, wdrożenie ≤30 dni.

### TRI-SEC-001 — przesyłanie tymczasowego hasła e-mailem

- **Aplikacja i komponent:** Trippics backend, onboarding federacyjny.
- **Kategoria:** hasła i odzyskiwanie konta.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `src/main/java/pl/trippics/service/impl/EmailServiceImpl.java:51-57` umieszcza tymczasowe hasło w treści wiadomości; `UserServiceImpl.java:156-174` generuje, zapisuje hash i wysyła hasło.
- **Wpływ:** hasło może pozostać w skrzynkach, kopiach poczty i systemach antyspamowych; użytkownik może nie zmienić go natychmiast.
- **Rozwiązanie:** nie wysyłać haseł; użyć jednorazowego, krótkotrwałego, zahashowanego tokenu „ustaw hasło”, z unieważnieniem po użyciu i rate limitingiem.
- **Właściciel:** Trippics Backend/Auth.
- **Termin:** ≤7 dni.

### MAT-RODO-001 — raporty awarii mogą utrwalać PII i sekrety

- **Aplikacja i komponent:** Mata24 mobile i backend.
- **Kategoria:** RODO / telemetryka / logowanie.
- **Severity:** High. **Confidence:** High. **Status:** Confirmed.
- **Dowód:** `mata24-mobile/src/lib/crashReporting.ts:16-30` wysyła pełny `message` i `stack`; `ClientErrorReportService.java:31-44` zapisuje je, wiąże z user ID, a komunikat ponownie zapisuje do logu. Nie znaleziono redakcji, zgody/opt-out ani retencji tej tabeli.
- **Wpływ:** błędy mogą zawierać tokeny, dane formularzy, identyfikatory i inne PII; dane są duplikowane w bazie i logach.
- **Rozwiązanie:** allow-list pól, redakcja tokenów/e-maili/URL query przed wysłaniem i ponownie na serwerze, nie logować pełnego komunikatu, ustalić krótki TTL i podstawę/cel, udokumentować telemetrykę oraz zapewnić opt-out, gdy wymagany.
- **Właściciel:** Mata24 Mobile/Backend + Privacy.
- **Termin:** ≤7 dni.

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
| Dostęp/poprawienie/przenoszenie/usunięcie | Gap | Edycja profilu obecna; brak eksportu/usunięcia konta. |
| Sprzeciw/ograniczenie | Needs evidence | Opisane w polityce, brak procesu technicznego. |
| Procesorzy/transfery | Gap | Kategorie ogólne i placeholder transferowy, bez listy rzeczywistych procesorów. |
| Privacy by design/default | Partial-positive | Ustawienia publiczności profilu i audyt zgód; należy potwierdzić bezpieczne defaulty live. |
| Naruszenia/DPIA | Needs evidence | Brak dowodu planu IR/DPIA. |
| Mobile permissions/telemetry | Not assessed | `yalquo-mobile` jest puste. |
| Usunięcie konta | Missing | YAL-RODO-002. |

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
