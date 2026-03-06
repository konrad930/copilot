# Plan implementacji funkcjonalności MVC

Jesteś doświadczonym architektem oprogramowania, którego zadaniem jest stworzenie szczegółowego planu wdrożenia nowej funkcjonalności w aplikacji ASP.NET MVC 5. Twój plan poprawi jakość kodu, ułatwi wdrożenie i zminimalizuje ryzyko błędów.

Zanim zaczniemy, zapoznaj się z poniższymi informacjami:

1. Jak powinien działać flow:
   <flow_steps>
      1) Użytkownik przechodzi na odpowiednią stronę (widok Razor) lub klika akcję w interfejsie.
      2) Kontroler MVC odbiera żądanie (GET/POST), waliduje dane wejściowe (ModelState).
      3) Logika biznesowa jest delegowana do odpowiedniego serwisu.
      4) Serwis wykonuje operacje na danych (EF6 / ADO.NET / LINQ to SQL), przetwarza logikę.
      5) Kontroler zwraca odpowiedni widok (View/PartialView/RedirectToAction) z modelem lub komunikatem.
      6) W przypadku operacji długotrwałych — zadanie jest delegowane do Quartz.NET lub SignalR informuje o postępie.
   </flow_steps>

2. Specyfikacja akcji kontrolera:
   <controller_action_specification>
      - Kontroler: [NazwaKontrolera]Controller
      - Akcja: [NazwaAkcji]
      - Metoda HTTP: GET / POST (lub obie — formularz + obsługa submit)
      - Route: [ścieżka URL, np. /Cases/{id}/Send, /Admin/Users/Edit/{id}]
      - Opis: [Co robi ta akcja — np. wyświetla formularz edycji użytkownika, przetwarza wysyłkę sprawy]
      - Autoryzacja: [Wymagane role/uprawnienia, np. [Authorize(Roles = "Admin")]]
      - Parametry wejściowe:
        - Route params: [np. int id]
        - Query string: [np. ?page=1&search=xyz]
        - Form data (POST): [np. ViewModel z formularza]
      - Wynik:
        - GET: View z modelem [NazwaViewModel]
        - POST sukces: RedirectToAction("[Akcja]") z TempData["SuccessMessage"]
        - POST błąd walidacji: ten sam View z ModelState errors
        - POST błąd biznesowy: View z komunikatem błędu lub redirect z TempData["ErrorMessage"]
   </controller_action_specification>

3. Powiązane zasoby bazy danych:
   <related_db_resources>
      {{entityClasses}} <- dodaj referencje do encji bazy danych powiązanych z tą funkcjonalnością (np. @entityClass1, @entityClass2)
   </related_db_resources>

4. Tech stack:
   <tech_stack>
      tech-stack.md
   </tech_stack>

5. Zasady implementacji:
   <implementation_rules>
      implementation-rules.md
   </implementation_rules>

6. Istniejące serwisy:
   <existing_services>
      SPService.cs
      UserService.cs
   </existing_services>

7. Istniejące widoki (opcjonalnie, jeśli modyfikujesz istniejący moduł):
   <existing_views>
      {{viewFiles}} <- dodaj referencje do istniejących widoków .cshtml, jeśli dotyczy
   </existing_views>

Twoim zadaniem jest stworzenie kompleksowego planu wdrożenia funkcjonalności w aplikacji MVC. Przed dostarczeniem ostatecznego planu użyj znaczników <analysis>, aby przeanalizować informacje.

1. Podsumuj kluczowe punkty specyfikacji akcji kontrolera.
2. Wymień wymagane i opcjonalne parametry wejściowe (route, query string, form data).
3. Wymień niezbędne typy: ViewModel, Command/DTO, walidatory.
4. Zaprojektuj ViewModel(e) — jakie pola, jakie atrybuty walidacji (DataAnnotations), jakie SelectList/dropdown dane.
5. Zastanów się, jak wyodrębnić logikę do serwisu (istniejącego lub nowego, jeśli nie istnieje).
6. Zaplanuj walidację danych:
   - Client-side: jQuery Validation / Unobtrusive Validation
   - Server-side: ModelState + walidacja biznesowa w serwisie
7. Zaprojektuj widoki Razor (.cshtml):
   - Layout, sekcje, partial views
   - Formularze: @Html.BeginForm, @Html.EditorFor, @Html.ValidationMessageFor
   - Wyświetlanie danych: tabele, listy, szczegóły
8. Określ sposób rejestrowania błędów w tabeli błędów (jeśli dotyczy).
9. Identyfikacja potencjalnych zagrożeń bezpieczeństwa:
   - AntiForgeryToken na formularzach POST
   - Autoryzacja (role, uprawnienia)
   - Ochrona przed mass assignment (Bind attribute / dedykowane ViewModele)
   - Encoding danych wyjściowych (XSS)
10. Nakreśl potencjalne scenariusze błędów i odpowiadające im strategie obsługi.
11. Jeśli funkcjonalność wymaga operacji asynchronicznych — zaplanuj integrację z Quartz.NET (job) lub SignalR (real-time feedback).
12. Zaplanuj rejestrację zależności w Ninject (nowe serwisy, repozytoria).

Po przeprowadzeniu analizy utwórz szczegółowy plan wdrożenia w formacie markdown. Plan powinien zawierać następujące sekcje:

1. Przegląd funkcjonalności
2. Szczegóły akcji kontrolera
3. ViewModele i typy danych
4. Widoki Razor
5. Przepływ danych
6. Walidacja (client-side + server-side)
7. Względy bezpieczeństwa
8. Obsługa błędów
9. Wydajność
10. Kroki implementacji

W całym planie upewnij się, że:

- Używać prawidłowego flow MVC:
  - GET → Kontroler pobiera dane → Buduje ViewModel → Zwraca View(viewModel)
  - POST → Kontroler odbiera formularz → Waliduje ModelState → Deleguje do serwisu → RedirectToAction (PRG pattern) lub zwraca View z błędami
- Stosować wzorzec PRG (Post-Redirect-Get) dla operacji POST, aby unikać podwójnego wysyłania formularzy.
- Używać TempData do przekazywania komunikatów sukcesu/błędu między redirect-ami.
- Stosować dedykowane ViewModele (nigdy nie przekazywać encji EF bezpośrednio do widoku).
- Używać @Html.AntiForgeryToken() w każdym formularzu POST i [ValidateAntiForgeryToken] na akcji.
- Stosować [Bind] lub dedykowane ViewModele, aby chronić przed mass assignment.
- Używać partial views (_PartialName.cshtml) dla reużywalnych komponentów UI.
- Jeśli akcja wymaga AJAX — używać $.ajax / fetch z PartialView lub JsonResult.
- Rejestrować nowe zależności w Ninject (NinjectWebCommon.cs lub odpowiednim module).
- Stosować odpowiednie kody odpowiedzi HTTP:
  - 200 (OK): Widok wyświetlony poprawnie.
  - 302 (Redirect): Po udanym POST — PRG pattern.
  - 400 (Bad Request): Błąd walidacji — zwrot widoku z ModelState errors.
  - 401 (Unauthorized): Brak uwierzytelnienia.
  - 403 (Forbidden): Brak uprawnień do operacji.
  - 404 (Not Found): Żądany zasób nie istnieje — return HttpNotFound().
  - 500 (Server Error): Błąd po stronie serwera — logowanie + przekierowanie na stronę błędu.
- Dostosowanie do dostarczonego stacku technologicznego.
- Postępuj zgodnie z podanymi zasadami implementacji.

Końcowym wynikiem powinien być dobrze zorganizowany plan wdrożenia w formacie markdown. Oto przykład tego, jak powinny wyglądać dane wyjściowe:

```markdown
# MVC Implementation Plan: [Nazwa funkcjonalności]

## 1. Przegląd funkcjonalności

[Krótki opis celu i funkcjonalności — co użytkownik zobaczy, co będzie mógł zrobić]

## 2. Szczegóły akcji kontrolera

- Kontroler: [Nazwa]Controller
- Akcje:
  - GET [Nazwa] — wyświetla [opis]
  - POST [Nazwa] — przetwarza [opis]
- Route: [wzorzec URL]
- Autoryzacja: [role/uprawnienia]
- Parametry:
  - Route: [parametry route]
  - Query string: [parametry opcjonalne]
  - Form (POST): [ViewModel]

## 3. ViewModele i typy danych

[Definicje ViewModeli z właściwościami i atrybutami walidacji DataAnnotations]
[Command Modele jeśli dotyczy]
[Mappingi: Entity ↔ ViewModel]

## 4. Widoki Razor

[Lista widoków .cshtml do stworzenia/modyfikacji]
[Opis layoutu, partial views, sekcji]
[Opis formularzy: pola, przyciski, walidacja client-side]
[Opis tabel/list jeśli dotyczy]

## 5. Przepływ danych

[Opis przepływu: Widok → Kontroler → Serwis → EF/DB → Serwis → Kontroler → Widok]
[Diagram jeśli potrzebny]

## 6. Walidacja

### Client-side
[jQuery Validation / Unobtrusive Validation — jakie reguły]

### Server-side
[ModelState validation — DataAnnotations]
[Walidacja biznesowa w serwisie — jakie reguły]

## 7. Względy bezpieczeństwa

[AntiForgeryToken, autoryzacja, mass assignment protection, XSS, encoding]

## 8. Obsługa błędów

[Lista potencjalnych błędów i sposób ich obsługi]
[Logowanie błędów]
[Komunikaty dla użytkownika]

## 9. Wydajność

[Potencjalne wąskie gardła i strategie optymalizacji]
[Eager loading vs lazy loading w EF]
[Caching jeśli dotyczy]

## 10. Kroki implementacji

1. [Krok 1 — np. Utworzenie ViewModelu]
2. [Krok 2 — np. Utworzenie/rozszerzenie serwisu]
3. [Krok 3 — np. Implementacja akcji kontrolera]
4. [Krok 4 — np. Utworzenie widoków Razor]
5. [Krok 5 — np. Rejestracja w Ninject]
6. [Krok 6 — np. Testy]
   ...
```

Końcowe wyniki powinny składać się wyłącznie z planu wdrożenia w formacie markdown i nie powinny powielać ani powtarzać żadnej pracy wykonanej w sekcji analizy.

Pamiętaj, aby zapisać swój plan wdrożenia w pliku {{FILE}}. Upewnij się, że plan jest szczegółowy, przejrzysty i zapewnia kompleksowe wskazówki dla zespołu programistów.