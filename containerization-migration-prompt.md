# Plan migracji i konteneryzacji aplikacji ASP.NET MVC — .NET Framework 4.8 → .NET 9.0 Core (Docker)

Jesteś doświadczonym architektem oprogramowania specjalizującym się w migracjach platformy .NET i konteneryzacji, którego zadaniem jest stworzenie szczegółowego planu migracji istniejącej bezstanowej aplikacji ASP.NET MVC 5 (.NET Framework 4.8) — konwertera XML na PDF — do ASP.NET Core MVC (.NET 9.0) z docelowym uruchomieniem w kontenerze Docker (Linux). Twój plan zapewni płynne przejście, zidentyfikuje wszystkie blokery konteneryzacji i zaproponuje zamienniki dla niekompatybilnych zależności.

Zanim zaczniemy, zapoznaj się z poniższymi informacjami:

## Kontekst migracji

Cel główny:
<migration_goal>
- Umieszczenie aplikacji w kontenerze Docker (Linux)
- Migracja z .NET Framework 4.8 → .NET 9.0 Core jako krok konieczny do konteneryzacji
- Aplikacja jest bezstanowa (konwerter XML → PDF), brak bazy danych, brak sesji
- Eliminacja zależności od IIS i Windows-only komponentów
</migration_goal>

Obecny stan aplikacji:
<current_application>
- Framework: .NET Framework 4.8
- Wzorzec: ASP.NET MVC 5
- Charakter: Bezstanowa — brak bazy danych, brak ORM, brak sesji
- Funkcja główna: Konwersja XML → PDF
- Hosting: IIS (System.Web pipeline) — Windows only
- Konfiguracja: Web.config (XML)
- IoC Container: Ninject
- Widoki: Razor (.cshtml) z @Html helpers
- Bundling: System.Web.Optimization (BundleConfig.cs)
- Biblioteki PDF: {{pdfLibrary}} <- np. iTextSharp, PdfSharp, Crystal Reports, RDLC, Telerik Reporting, DevExpress, FOP, wkhtmltopdf, inne — WYMIEŃ WSZYSTKIE
- Parsowanie XML: {{xmlLibrary}} <- System.Xml / System.Xml.Linq / XslCompiledTransform / Saxon / inne
- Zależności natywne / Windows-only: {{nativeDependencies}} <- np. COM components, GDI+ (System.Drawing), Windows Registry, GAC assemblies, Windows fonts, natywne DLL-ki (C++/CLI), oleaut32.dll, inne
- Templating PDF: {{pdfTemplatingApproach}} <- np. XSLT → FO → PDF, HTML → PDF, szablon .rdlc, Crystal Reports .rpt, bezpośrednie generowanie z kodu, inne
- Upload plików: HttpPostedFileBase (System.Web)
- Autentykacja: {{authMethod}} <- Windows Auth / Forms Auth / Anonymous / inne
</current_application>

Docelowy stan aplikacji:
<target_application>
- Framework: .NET 9.0
- Wzorzec: ASP.NET Core MVC
- Charakter: Bezstanowa — bez bazy danych
- Hosting: Kestrel w kontenerze Docker (Linux-based image)
- Obraz bazowy: mcr.microsoft.com/dotnet/aspnet:9.0 (Debian-based) lub Alpine
- IoC Container: Wbudowany DI (Microsoft.Extensions.DependencyInjection)
- Konfiguracja: appsettings.json + Environment Variables (Docker env / secrets)
- Biblioteki PDF: {{targetPdfLibrary}} <- zamiennik kompatybilny z Linux/.NET 9 — DO USTALENIA W ANALIZIE
- Widoki: Razor z Tag Helpers
- Bundling: LibMan / Vite / brak (aplikacja nie wymaga rozbudowanego frontu)
</target_application>

## Pliki źródłowe do analizy

Struktura obecnego projektu:
<project_structure>
{{solutionFile}} <- referencja do pliku .sln
{{csprojFiles}} <- referencje do plików .csproj (stary format)
{{packagesConfig}} <- packages.config LUB PackageReference w .csproj
</project_structure>

Konfiguracja:
<configuration_files>
{{webConfig}} <- Web.config (główny) — SZCZEGÓLNIE WAŻNE: sekcje system.web, httpRuntime, appSettings, assemblyBinding
{{webConfigTransforms}} <- Web.Debug.config, Web.Release.config
{{bundleConfig}} <- BundleConfig.cs
{{routeConfig}} <- RouteConfig.cs
{{filterConfig}} <- FilterConfig.cs
{{ninjectConfig}} <- NinjectWebCommon.cs / moduły Ninject
</configuration_files>

Obecny tech stack:
<tech_stack>
tech-stack.md
</tech_stack>

Zasady implementacji:
<implementation_rules>
implementation-rules.md
</implementation_rules>

Kontrolery do migracji:
<controllers>
{{controllerFiles}} <- referencje do plików kontrolerów
</controllers>

Serwisy do migracji:
<existing_services>
{{serviceFiles}} <- referencje do serwisów konwersji XML→PDF
</existing_services>

Logika konwersji XML → PDF:
<conversion_logic>
{{xmlParsingFiles}} <- pliki parsowania XML
{{xsltFiles}} <- pliki XSLT transformacji (jeśli używane)
{{pdfGenerationFiles}} <- pliki generowania PDF
{{templateFiles}} <- szablony PDF / XSLT / HTML / RDLC / inne
</conversion_logic>

Natywne i zewnętrzne zależności (KRYTYCZNE DLA KONTENERYZACJI):
<native_dependencies>
{{nativeDlls}} <- referencje do natywnych DLL-ek w projekcie (np. bin/*.dll, lib/*.dll) — DLL-ki nie będące pakietami NuGet
{{comReferences}} <- referencje COM w .csproj (np. <COMReference>)
{{gacAssemblies}} <- assemblies z GAC, które nie są NuGet packages
{{thirdPartyTools}} <- zewnętrzne narzędzia wywoływane z kodu (np. wkhtmltopdf.exe, Apache FOP, Prince XML)
{{referencedDlls}} <- referencje <Reference> w .csproj wskazujące na lokalne .dll (nie NuGet)
</native_dependencies>

Widoki do migracji:
<existing_views>
{{viewFiles}} <- referencje do istniejących widoków .cshtml
{{layoutFile}} <- _Layout.cshtml
</existing_views>

## Instrukcje analizy

Twoim zadaniem jest stworzenie kompleksowego planu migracji i konteneryzacji. Przed dostarczeniem ostatecznego planu użyj znaczników `<analysis>`, aby przeanalizować informacje.

### FAZA 1: Audyt blokerów konteneryzacji (NAJWAŻNIEJSZE)

1. **Audyt WSZYSTKICH zależności projektu** — przejdź przez `.csproj` i `packages.config` i sklasyfikuj KAŻDĄ zależność:

   | Kategoria | Przykłady | Ryzyko konteneryzacji |
   |---|---|---|
   | NuGet — kompatybilny z .NET 9 Linux | Newtonsoft.Json, System.Xml.Linq | ✅ Brak ryzyka |
   | NuGet — wymaga upgrade do wersji Core | iTextSharp → iText 8 | ⚠️ Wymaga pracy |
   | NuGet — brak wersji Core, istnieje zamiennik | PdfSharp → PdfSharpCore/QuestPDF | ⚠️ Wymaga pracy |
   | NuGet — brak wersji Core, brak zamiennika | Crystal Reports SDK | 🔴 BLOKER |
   | Lokalna DLL — managed .NET Framework only | CustomLib.dll (target: net48) | 🔴 BLOKER |
   | Lokalna DLL — natywna Windows (C++/CLI, COM) | pdfium.dll, oleaut32.dll | 🔴 BLOKER |
   | COM Reference | Microsoft.Office.Interop.* | 🔴 BLOKER |
   | GAC Assembly | nie dostępny przez NuGet | 🔴 BLOKER |
   | Zewnętrzne .exe narzędzia | wkhtmltopdf.exe, fop.bat | ⚠️ Wymaga zamiennika lub instalacji w kontenerze |
   | System.Drawing / GDI+ | System.Drawing.Common | ⚠️ Wymaga libgdiplus na Linux lub zamiennik |
   | Windows Registry access | Registry.GetValue() | 🔴 BLOKER |
   | Windows-specific paths | `C:\Fonts\`, `C:\Templates\` | ⚠️ Wymaga refaktoru |

2. **Głęboka analiza biblioteki PDF** — to jest KRYTYCZNY element:
   - Jaka biblioteka/podejście jest obecnie używane do generowania PDF?
   - Czy biblioteka ma wersję kompatybilną z .NET 9 + Linux?
   - Czy korzysta z GDI+ / System.Drawing (problem na Linux)?
   - Czy korzysta z natywnych DLL-ek Windows?
   - Czy korzysta z COM (np. Office Interop do konwersji)?
   - Czy wymaga zainstalowanych fontów Windows (Arial, Times New Roman, Calibri)?
   - Czy PDF output zależy od fontów systemowych (embedding vs referencing)?
   - Zaproponuj KONKRETNY zamiennik z uzasadnieniem:
     - **QuestPDF** — nowoczesny, fluent API, .NET native, Linux-friendly, MIT license
     - **iText 8** — enterprise, AGPL (uwaga licencja!), Linux-friendly
     - **PdfSharpCore** — community fork PdfSharp dla .NET Core
     - **Aspose.PDF** — komercyjny, Linux-friendly, cross-platform
     - **wkhtmltopdf** — można zainstalować w kontenerze Linux (apt-get)
     - **Puppeteer Sharp / Playwright** — headless Chrome/Chromium do HTML→PDF
     - **Inne** — w zależności od obecnego rozwiązania

3. **Głęboka analiza szablonów / template'ów PDF**:
   - Czy PDF jest generowany z szablonu (XSLT → FO → PDF, RDLC, Crystal Reports, HTML template)?
   - Czy szablony zawierają odwołania do Windows-specific zasobów (fonty, ścieżki, registry)?
   - Czy szablony są kompilowane (RDLC/Crystal) czy interpretowane (XSLT/HTML)?
   - Jaki jest nakład pracy na migrację szablonów na nowe rozwiązanie?
   - Czy możliwe jest zachowanie obecnych szablonów z nową biblioteką, czy trzeba je przepisać?

4. **Analiza XSLT (jeśli dotyczy)**:
   - `XslCompiledTransform` — kompatybilność z .NET Core:
     - ✅ Podstawowe transformacje XSLT 1.0 — działają
     - ⚠️ `msxsl:script` (embedded scripting) — NIE DZIAŁA w .NET Core
     - ⚠️ `document()` function — może wymagać konfiguracji `XsltSettings`
     - ⚠️ Extension objects — mogą wymagać zmian
   - Czy XSLT pliki używają `msxsl:script`? → Jeśli tak, trzeba wyekstrahować logikę do C#
   - Czy XSLT produkuje XSL-FO? → Jeśli tak, czym jest renderowany FO → PDF? (Apache FOP? .NET FO library?)

5. **Analiza fontów (KRYTYCZNE dla PDF w kontenerze Linux)**:
   - Jakie fonty są używane w generowanym PDF?
   - Czy fonty Windows (Arial, Times New Roman, Calibri, Verdana) są wymagane?
   - Czy fonty są embeddowane w PDF czy referencjonowane?
   - Plan dostarczenia fontów w kontenerze:
     - Instalacja pakietu `ttf-mscorefonts-installer` w Dockerfile
     - Kopiowanie plików .ttf do kontenera
     - Zmiana na fonty open-source (Liberation, DejaVu, Noto)
     - Embedding fontów w aplikacji (jako embedded resources)

6. **Analiza System.Drawing / GDI+**:
   - Czy kod korzysta z `System.Drawing`, `System.Drawing.Common`, `Bitmap`, `Graphics`, `Image`?
   - Na Linux `System.Drawing.Common` wymaga `libgdiplus` (niestabilne, deprecated w .NET 7+)
   - Zamienniki:
     - **SkiaSharp** — cross-platform 2D graphics (Google Skia)
     - **ImageSharp** (SixLabors) — managed .NET, brak natywnych zależności
     - **Magick.NET** — ImageMagick wrapper

### FAZA 2: Analiza migracji kodu

7. **Analiza breaking changes .NET Framework → .NET Core**:
   - `System.Web.*` → `Microsoft.AspNetCore.*`
   - `HttpContext.Current` → `IHttpContextAccessor` (lub eliminacja — w bezstanowej aplikacji często niepotrzebne)
   - `HttpPostedFileBase` → `IFormFile`
   - `Server.MapPath("~/Templates")` → `IWebHostEnvironment.ContentRootPath` + `Path.Combine`
   - `ConfigurationManager.AppSettings["key"]` → `IConfiguration` / `IOptions<T>`
   - `Response.BinaryWrite` / `Response.OutputStream` → `FileContentResult` / `FileStreamResult`
   - `@Html.BeginForm` → `<form asp-action="">` Tag Helpers
   - `BundleConfig.cs` → usunąć (minimalistyczny front)
   - `Global.asax` → `Program.cs`
   - `RouteConfig.cs` → `MapControllerRoute` / Attribute Routing
   - `FilterConfig.cs` → Middleware + rejestracja filtrów w `Program.cs`
   - `Web.config appSettings` → `appsettings.json` → `IOptions<T>` pattern
   - `Web.config assemblyBinding` → usunąć (nie dotyczy .NET Core)
   - `Web.config httpRuntime maxRequestLength` → Kestrel limits w `Program.cs`

8. **Analiza DI — Ninject → wbudowany DI**:
   - Zmapuj KAŻDĄ rejestrację Ninject na wbudowany DI
   - Określ lifecycle: Transient / Scoped / Singleton
   - Serwisy konwersji XML→PDF — zazwyczaj Transient lub Scoped (bezstanowe)

9. **Analiza widoków Razor**:
   - `@Html.*` helpers → Tag Helpers
   - `@Scripts.Render` / `@Styles.Render` → bezpośrednie `<script>` / `<link>` (aplikacja ma minimalistyczny front)
   - Formularz uploadu XML: `enctype="multipart/form-data"` + `<input asp-for="" type="file">`
   - Czy front jest w ogóle potrzebny? Może wystarczy API endpoint (Controller zwracający FileResult)?

10. **Analiza security**:
    - **XXE Protection** (XML External Entity) — KRYTYCZNE:
      ```csharp
      var settings = new XmlReaderSettings {
          DtdProcessing = DtdProcessing.Prohibit,
          XmlResolver = null
      };
      ```
    - Czy obecny kod ma ochronę przed XXE? Jeśli nie — dodać przy migracji.
    - AntiForgeryToken — migracja do Core (automatyczny w `<form>` z Tag Helpers)
    - Walidacja uploadu: rozmiar, typ MIME, rozszerzenie, zawartość
    - HTTPS enforcement w kontenerze

### FAZA 3: Projekt konteneryzacji

11. **Projekt Dockerfile**:
    - Obraz bazowy: `mcr.microsoft.com/dotnet/aspnet:9.0` (Debian) vs `mcr.microsoft.com/dotnet/aspnet:9.0-alpine` (mniejszy, ale mniej pakietów)
    - Multi-stage build: `sdk:9.0` → build, `aspnet:9.0` → runtime
    - Instalacja dodatkowych pakietów systemowych (fonty, libgdiplus, wkhtmltopdf — w zależności od analizy)
    - Kopiowanie fontów / szablonów / zasobów do kontenera
    - Konfiguracja użytkownika non-root
    - Healthcheck endpoint
    - Porty: EXPOSE 8080 (domyślny w .NET 8+)

12. **Konfiguracja kontenerowa**:
    - Environment variables zamiast appsettings per-environment
    - Docker secrets dla wrażliwych danych (jeśli dotyczy)
    - Volume mounts — czy potrzebne? (pliki tymczasowe, logi)
    - Resource limits (memory, CPU) — ważne dla operacji generowania PDF (memory-intensive)
    - Tmp directory — upewnij się, że kontener ma writeable `/tmp`

13. **Docker Compose / orchestracja** (jeśli dotyczy):
    - Czy aplikacja wymaga dodatkowych serwisów (np. Apache FOP jako osobny kontener)?
    - Reverse proxy (nginx/traefik) przed aplikacją?
    - Logging driver (stdout/stderr → Docker logs → ELK/Loki)

14. **Strategia migracji** — określ podejście:
    - **Rekomendowane dla bezstanowej aplikacji**: Side-by-side (nowy projekt .NET 9 Core, przenoszenie kodu z refaktorem)
    - Uzasadnienie: bezstanowa aplikacja = mało kodu do przeniesienia, czysta migracja lepsza niż upgrade-in-place
    - Ile kodu można przenieść 1:1?
    - Ile wymaga przepisania (głównie: biblioteka PDF, szablony)?

15. **Identyfikacja ryzyk i blokerów**:

    | Ryzyko | Prawdopodobieństwo | Wpływ | Mitygacja |
    |---|---|---|---|
    | Biblioteka PDF niekompatybilna z Linux/.NET 9 | ? | 🔴 Krytyczny | Zamiennik: [konkretna propozycja] |
    | Różnice w renderowaniu PDF (fonty, layout) | Wysokie | 🔴 Krytyczny | Testy porównawcze pixel-to-pixel |
    | XSLT z msxsl:script | ? | 🟡 Średni | Ekstrakcja logiki do C# |
    | System.Drawing na Linux | ? | 🟡 Średni | SkiaSharp / ImageSharp |
    | Natywne DLL-ki Windows | ? | 🔴 Krytyczny | Zamiennik managed / Linux native |
    | Brak fontów Windows w kontenerze | Wysokie | 🔴 Krytyczny | Instalacja fontów w Dockerfile |
    | COM References | ? | 🔴 Krytyczny | Eliminacja / zamiennik |
    | Limity pamięci kontenera przy dużych PDF | Średnie | 🟡 Średni | Resource limits, streaming |

## Format wyjściowy

Po przeprowadzeniu analizy utwórz szczegółowy plan migracji w formacie markdown. Plan powinien zawierać następujące sekcje:

```markdown
# Plan migracji i konteneryzacji: [Nazwa Aplikacji] — .NET 4.8 → .NET 9.0 Docker

## 1. Przegląd i cele

[Cel: konteneryzacja aplikacji]
[Migracja .NET 4.8 → .NET 9.0 Core jako krok konieczny]
[Charakterystyka: bezstanowy konwerter XML → PDF]
[Wybrana strategia migracji — z uzasadnieniem]

## 2. Audyt zależności i blokery konteneryzacji

### 2.1 Klasyfikacja wszystkich zależności
| Zależność | Typ | Obecna wersja | Kompatybilność .NET 9 Linux | Status | Zamiennik / Akcja |
|---|---|---|---|---|---|
| [każda zależność] | NuGet/DLL/COM/GAC/exe | x.x | ✅/⚠️/🔴 | OK/Upgrade/Zamiennik/BLOKER | [co zrobić] |

### 2.2 Blokery konteneryzacji (KRYTYCZNE)
[Lista elementów, które UNIEMOŻLIWIAJĄ uruchomienie w kontenerze Linux]
[Dla każdego blokera: opis problemu + proponowane rozwiązanie]

### 2.3 Elementy wymagające pracy (WARNING)
[Lista elementów wymagających zmian, ale nie blokujących]

## 3. Migracja pipeline konwersji XML → PDF

### 3.1 Obecny flow konwersji
[Szczegółowy opis: XML input → [kroki pośrednie] → PDF output]
[Diagram przepływu]
[Użyte biblioteki na każdym etapie]

### 3.2 Analiza biblioteki PDF
[Obecna biblioteka — szczegóły]
[Kompatybilność z .NET 9 / Linux — wynik analizy]
[Zależności natywne (GDI+, DLL, COM) — wynik analizy]
[REKOMENDACJA: zamiennik lub upgrade — z uzasadnieniem]
[Nakład pracy na migrację]

### 3.3 Analiza szablonów / template'ów
[Obecny mechanizm szablonów]
[Kompatybilność z docelowym rozwiązaniem]
[Czy szablony trzeba przepisać? Nakład pracy?]

### 3.4 Analiza fontów
[Wymagane fonty]
[Strategia dostarczenia fontów w kontenerze Linux]
[Testy renderowania z fontami Linux]

### 3.5 Docelowy flow konwersji
[Nowy pipeline po migracji]
[Nowe biblioteki na każdym etapie]
[Różnice vs obecny flow]

## 4. Migracja struktury projektu

### 4.1 Nowy .csproj (SDK-style)
[Docelowy format pliku projektu]
[Target Framework: net9.0]
[PackageReferences]

### 4.2 Program.cs
[Konfiguracja middleware pipeline]
[Rejestracja DI]
[Kestrel limits (upload size)]
[HTTPS, HSTS]

### 4.3 appsettings.json
[Mapowanie z Web.config]
[Konfiguracja specyficzna dla kontenera (env vars)]

## 5. Migracja DI (Ninject → wbudowany DI)

| Rejestracja Ninject | Odpowiednik MS DI | Lifecycle | Uwagi |
|---|---|---|---|
| [każda rejestracja] | [odpowiednik] | [lifecycle] | [uwagi] |

## 6. Migracja kontrolerów

[Zmiana klasy bazowej]
[HttpPostedFileBase → IFormFile]
[FileResult dla PDF download]
[Server.MapPath → IWebHostEnvironment]
[ConfigurationManager → IOptions<T>]

## 7. Migracja widoków Razor

[Tag Helpers zamiast @Html helpers]
[Usunięcie BundleConfig]
[Formularz uploadu z enctype multipart]
[Minimalistyczny front — czy w ogóle potrzebny? Może API-only?]

## 8. Konteneryzacja

### 8.1 Dockerfile
[Multi-stage build]
[Obraz bazowy — wybór i uzasadnienie]
[Instalacja dodatkowych pakietów (fonty, narzędzia)]
[Kopiowanie zasobów (szablony, fonty)]
[Non-root user]
[Healthcheck]

### 8.2 .dockerignore
[Wykluczenia]

### 8.3 Docker Compose (jeśli potrzebny)
[Definicja serwisów]
[Volumes, environment, ports]

### 8.4 Konfiguracja kontenerowa
[Environment variables]
[Resource limits (pamięć — ważne dla PDF generation)]
[Logging (stdout/stderr)]
[Tmp directory]

## 9. Bezpieczeństwo

[XXE Protection — DtdProcessing.Prohibit, XmlResolver = null]
[Walidacja uploadu XML (rozmiar, typ, zawartość)]
[AntiForgeryToken (automatyczny w Core)]
[HTTPS w kontenerze]
[Non-root user w Dockerfile]
[Brak wrażliwych danych w WARSTWIE obrazu Docker (Docker secrets / env vars)]
[Content-Security-Policy]
[Request size limits (Kestrel)]

## 10. Obsługa błędów i logowanie

[UseExceptionHandler / UseStatusCodePages]
[ILogger<T> → stdout (Docker logs)]
[Structured logging (Serilog / NLog — opcjonalnie)]
[Błędy parsowania XML → 400 Bad Request z opisem]
[Błędy generowania PDF → 500 z logowaniem szczegółów]
[Health check endpoint → /health]
[Kody HTTP: 200, 400, 413, 415, 500]

## 11. Wydajność i limity w kontenerze

[Memory limits — generowanie PDF jest memory-intensive]
[Streaming dużych plików zamiast buforowania w pamięci]
[Async/await]
[Concurrent request limits (kontener ma ograniczone zasoby)]
[Graceful shutdown (SIGTERM handling)]
[Temp file cleanup]

## 12. Testy

### 12.1 Testy porównawcze PDF (KRYTYCZNE)
[Ten sam XML → stary pipeline vs nowy pipeline → porównanie PDF]
[Visual regression: pixel-to-pixel comparison]
[Porównanie rozmiaru pliku, struktury, fontów]
[Zbiór testowych XMLi pokrywający wszystkie case'y]

### 12.2 Testy integracyjne
[WebApplicationFactory<Program>]
[Upload XML → konwersja → weryfikacja PDF response]
[Testy w kontenerze (docker run → test)]

### 12.3 Testy edge cases
[Pusty XML, XML z błędami, za duży plik, złośliwy XML (XXE), nieprawidłowy format]

## 13. CI/CD i deployment

[Build pipeline: dotnet restore → build → test → docker build → push]
[Container registry (ACR, ECR, Docker Hub, GHCR)]
[Deployment target: Kubernetes / Docker Compose / Azure Container Apps / ECS — do określenia]
[Health checks i readiness probes]
[Rolling update strategy]

## 14. Kroki implementacji (kolejność)

### Etap 0: Audyt i Proof of Concept
1. [Audyt wszystkich zależności — klasyfikacja wg tabeli z sekcji 2]
2. [Identyfikacja blokerów konteneryzacji]
3. [PoC: uruchomienie pipeline konwersji XML→PDF na .NET 9 Linux (poza kontenerem)]
4. [PoC: porównanie PDF output — stary vs nowy pipeline]
5. [DECYZJA GO/NO-GO na podstawie PoC]

### Etap 1: Migracja projektu
6. [Utworzenie nowego projektu ASP.NET Core MVC (.NET 9.0) — SDK-style .csproj]
7. [Konfiguracja Program.cs (middleware, DI, Kestrel)]
8. [Migracja appsettings.json z Web.config]

### Etap 2: Migracja logiki konwersji
9. [Migracja/zamiana biblioteki PDF]
10. [Migracja parsowania XML]
11. [Migracja szablonów/template'ów (jeśli wymagana)]
12. [Dostarczenie fontów (pliki .ttf w projekcie)]
13. [Testy porównawcze PDF]

### Etap 3: Migracja MVC
14. [Migracja ViewModeli]
15. [Migracja kontrolerów (IFormFile, FileResult)]
16. [Migracja widoków Razor (Tag Helpers)]
17. [Migracja DI (Ninject → MS DI)]

### Etap 4: Konteneryzacja
18. [Stworzenie Dockerfile (multi-stage)]
19. [Stworzenie .dockerignore]
20. [Instalacja fontów i zależności w Dockerfile]
21. [Testy w kontenerze]
22. [Stworzenie Docker Compose (jeśli potrzebny)]

### Etap 5: Finalizacja
23. [Security hardening (XXE, non-root, HTTPS)]
24. [Konfiguracja logowania (stdout)]
25. [Health check endpoint]
26. [CI/CD pipeline (build → test → docker build → push)]
27. [Testy end-to-end w kontenerze]
28. [Dokumentacja]
29. [Deployment na środowisko docelowe]
```

## Zasady ogólne

W całym planie upewnij się, że:

- **Konteneryzacja jest celem nadrzędnym** — każda decyzja powinna być oceniana przez pryzmat: "czy to zadziała w kontenerze Linux?"
- **Brak bazy danych** — aplikacja jest bezstanowa, nie planujemy żadnej migracji ORM/bazy.
- **Pipeline konwersji XML → PDF to serce aplikacji** — największy wysiłek to migracja/zamiana bibliotek PDF i szablonów.
- **Fonty to ukryty bloker** — wiele bibliotek PDF wymaga fontów systemowych, których brak na Linux. Zaplanuj dostarczenie fontów.
- **System.Drawing / GDI+ nie działa na Linux** — jeśli biblioteka PDF z tego korzysta, to jest BLOKER. Zaproponuj zamiennik.
- **Native DLL-ki Windows nie działają na Linux** — jeśli projekt referencuje natywne DLL-ki, to jest BLOKER.
- **COM nie działa na Linux** — jeśli projekt używa COM, to jest BLOKER. Zaproponuj zamiennik.
- **PoC przed pełną migracją** — najpierw uruchom sam pipeline konwersji na .NET 9 Linux, dopiero potem migruj resztę.
- **Testy porównawcze PDF są obowiązkowe** — wygenerowany PDF MUSI wyglądać tak samo jak ze starego systemu.
- **Ochrona przed XXE** — `DtdProcessing = DtdProcessing.Prohibit`, `XmlResolver = null`.
- **Docker best practices** — multi-stage build, non-root user, .dockerignore, health checks.
- **PRG pattern** — dla formularzy POST (upload XML).
- **Streaming** — dla dużych plików XML/PDF preferować streaming.
- Stosować odpowiednie kody HTTP: 200, 302, 400, 401, 403, 404, 413, 415, 500.
- Dostosowanie do dostarczonego stacku technologicznego.
- Postępuj zgodnie z podanymi zasadami implementacji.

Końcowym wynikiem powinien być dobrze zorganizowany plan migracji i konteneryzacji w formacie markdown.

Pamiętaj, aby zapisać swój plan w pliku {{FILE}}. Upewnij się, że plan jest szczegółowy, przejrzysty i zapewnia kompleksowe wskazówki dla zespołu programistów.