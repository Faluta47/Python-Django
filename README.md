# Biblioteca Startup - README

Acest README explică complet proiectul „Biblioteca Startup” — ce tehnologii folosesc, arhitectura, cum rulezi, cum folosești aplicația ca user/administrator, detalii despre modelul de date, logica de împrumut/rezerve, sistemul de taxe pentru restanțe și testele automate.

---

Cuprins

- Descriere generală
- Stack tehnic
- Structura proiectului
- Modele (models)
- Routing și Views
- Template-uri și Frontend
- JavaScript (comportamente dinamice)
- Sistem de taxe pentru restanțe (fine)
- Cum rulezi aplicația local
- Cum folosești aplicația (client / admin) — pași detaliați
- Mentenanță și cum modifici setările (ex: valoarea taxei)
- Testare automată
- Probleme cunoscute și debugging
- Contribuire

---

Descriere generală

Aplicația este un prototip de bibliotecă online cu funcționalități esențiale:
- catalog de cărți
- împrumuturi cu limită (max 3 active)
- returnare cu restabilire de stoc
- rezervări (pending/active/picked_up/cancelled/expired)
- pagină de profil pentru membri (statistici, activitate, taxe)
- interfață pentru bibliotecari (lista clienților, detaliu client, gestionare rezervări)
- sistem de taxe pentru întârzieri (configurabil)

Stack tehnic

- Backend: Django 6.x (Python)
- ORM/models: Django models
- Templates: Django Templates + Bootstrap 5 (CSS/JS via CDN)
- Frontend behavior: Vanila JS (fetch API) pentru acțiuni AJAX în admin
- Baza de date: SQLite (implicit în proiect)
- Teste: Django Test framework (unittests)

Structura proiectului (importantă)

- `manage.py` — utilitar Django
- `biblioteca_core/` — setări, urls, wsgi
- `management/` — aplicația principală
  - `models.py` — modele (Carte, Imprumut, Rezervare)
  - `views.py` — logica aplicației
  - `templates/` — template-urile (acasa, profil, lista_clienti, partials etc.)
  - `forms.py` — formulare (CarteForm, UserEditForm)
  - `tests.py` — teste automate

Modele (schemă și comportament)

- Carte
  - titlu, autor, isbn, categorie, numar_exemplare
  - `clean()` previne numar_exemplare negativ

- Imprumut
  - carte (FK), membru (FK catre User), data_imprumut, scadenta, returnat
  - la creare, `scadenta = date.today() + 14 zile`
  - metode adăugate:
    - `overdue_days()` — returnează numărul de zile întârziere (0 dacă nu e restanță)
    - `fine_amount(per_day=None)` — returnează Decimal taxă bazată pe zilele întârziere; folosește `settings.FINE_PER_DAY` dacă `per_day` nu e specificat

- Rezervare
  - carte, membru, data_rezervarii, data_expirarii (7 zile), status (pending/active/...)\

Routing și Views (fluxuri relevante)

- `acasa` (`/`) — dashboard pentru anonimi, membri și staff (diferit pentru fiecare)
  - pentru membri: afișează împrumuturi active, scadențe, și acum și total taxe `total_fines` și detaliu taxe per împrumut
- `lista_carti` (`/carti/`) — listare cărți
- `imprumuta/<id>/` — creare împrumut (login required)
- `returneaza/<id>/` — marcarea împrumutului ca returnat (staff only via view decorator)
- `rezerva/<id>/` & `lista_rezervari` — rezervări
- `lista_clienti` (`/clienti/`) — listă membrii (staff) + panel AJAX pentru detaliu client
- `clienti/<id>/` — returnează partial HTML (admin panel) cu detalii client
- `clienti/<id>/edit/` — POST pentru edit client (AJAX)
- `profil/` — pagina de profil a utilizatorului (member)
- `profil/setari/` — pagina de setări editabile de către utilizator

Template-uri și Frontend

- Template-uri principale în `management/templates/`:
  - `base.html` — include Bootstrap (CDN), navbar și blocul `{% block content %}`
  - `acasa.html` — dashboard (conține logică diferită pentru anonimi, membri, staff)
  - `profil.html` — profil utilizator (info, ultimele activități și taxe)
  - `lista_clienti.html` — listă clienți cu panel AJAX pentru detaliu
  - `partials/_user_panel.html` — partial folosit pentru detaliu client (folosit de staff)
  - `profil_setari.html` — form pentru edit profil (user)

JavaScript (comportamente dinamice)

- `lista_clienti.html` conține un script vanilla JS care:
  - face fetch AJAX la `/clienti/<id>/` la click pe un utilizator
  - inserează HTML-ul partial în panelul din dreapta
  - leagă butoanele Edit/Delete din panel pentru a face requesturi POST (folosind token CSRF din cookie)

Am reparat problema „Butonul Edit nu face nimic” astfel:
- Cauza: formularul de edit se află în tab-ul `Împrumuturi` (loans) care este inițial ascuns; butonul `Editează` e în headerul card-ului iar când utilizatorul apasă, JS făcea `form.style.display='block'` dar dacă tab-ul respectiv era ascuns utilizatorul nu vedea nimic.
- Soluția: la click pe `Editează` scriptul activează tab-ul `Împrumuturi` (folosind API-ul Bootstrap Tab sau un fallback) și apoi afișează formularul, plus dă focus pe primul input.
- Am aplicat fixul în `management/templates/lista_clienti.html`.

Sistem de taxe (fines)

- Implementare:
  - valoarea per zi e definită în `biblioteca_core/settings.py` ca `FINE_PER_DAY = Decimal('10.00')`
  - `Imprumut.overdue_days()` calculează zilele întârzierii
  - `Imprumut.fine_amount(per_day=None)` calculează suma (Decimal)
  - View-urile (`acasa`, `profil`, `lista_clienti`, `client_detail` partial) colectează taxele și le transmit template-urilor:
    - `total_fines` — suma totală pentru utilizator
    - `fines_map` — dict mapping `imprumut.id -> fine Decimal` (sau setez `imp.fine_amount_value` direct pe obiect pentru template)
- Afișare:
  - dashboard member: banner cu total taxe
  - profil member: afiș total taxe și taxa per carte (acolo unde e restanță)
  - lista clienți: lângă fiecare membru am afișat `u.total_fines` (dacă > 0)
  - partial admin client: arată taxa totală și taxa per împrumut alături de buton `Returnează`

Cum rulezi aplicația local (setup minimal)

1. Creează/activează virtualenv:
```bash
python3 -m venv venv
source venv/bin/activate
```
2. Instalează dependențele (proiectul folosește doar Django). Dacă vrei să fixezi versiunea, rulează:
```bash
pip install django==6.0.7
```
3. Migrează baza de date:
```bash
python manage.py migrate
```
4. Creează un superuser (opțional, pentru admin Django):
```bash
python manage.py createsuperuser
```
5. Rulează serverul:
```bash
python manage.py runserver
```
6. Accesează aplicația la: http://127.0.0.1:8000/

Cum folosești aplicația — ghid pas cu pas

Ca utilizator (client):
- Înregistrează-te (/inregistrare) sau autentifică-te (/accounts/login/)
- Vezi catalogul la `/carti/` și apasă „Împrumută” pentru a împrumuta (max 3 active)
- Revino la `Profil` pentru a vedea ce ai împrumutat, scadențele și eventualele taxe pentru restanțe
- Dacă ai restanțe, vezi suma totală pe dashboard și detaliile per carte
- Poți anula/rezerva cărți din catalog

Ca bibliotecar (admin intern):
- Autentifică-te cu cont staff (is_staff=True)
- Mergi la `Clienți` -> vei vedea listă cu toți membrii; fă click pe un membru pentru a vedea panelul cu detalii (partial AJAX)
- În panel poți edita datele membrului (buton `Editează`), șterge membrul (`Șterge`) sau returna o carte (buton `Returnează`) — toate operațiunile folosesc fetch/AJAX
- Vei vedea în listă valorile taxelor pentru membrii cu restanțe

Reparație buton Edit (detalii)

- Am fixat problema prin schimbarea scriptului din `lista_clienti.html` astfel încât la click pe `Editează` se activează tab-ul corespunzător (unde e ascuns formularul) și apoi se afișează formularul. Acum butonul funcționează și în cazul în care formularul era într-un tab ascuns.

Cum modific valoarea taxei (fine per day)

- Deschis `biblioteca_core/settings.py` și modifică `FINE_PER_DAY = Decimal('10.00')` la orice valoare dorești (ex. `Decimal('5.00')`).
- Recomandare: în producție, expune această valoare într-un sistem de configurare (variabilă de mediu) sau model configurabil.

Testare automată

- Toate testele aplicatiei se află în `management/tests.py`.
- Am adăugat teste care acoperă fluxurile:
  - vizitare homepage anonim
  - acces profil, setări profil
  - împrumut și returnare
  - rezervare
  - CRUD client (adăugare, editare, ștergere)
  - calcul taxe pentru restanțe (vizibil pentru utilizator și admin)

Rulează testele:
```bash
python manage.py test management -v 2
```

Rezultatul a fost: toate testele trec în mediul meu de dezvoltare.

Probleme cunoscute / sugestii pentru producție

- Nu există flux de plată: taxele sunt calculate din restanțe active; nu există un model pentru plăți. În producție ar trebui adăugat model `Payment` și un API/flux pentru a marca taxele ca plătite.
- Fallback JavaScript: dacă JS este dezactivat, anumite acțiuni nu vor funcționa (formularul de editare al clientului folosește AJAX). Am implementat un fix pentru a face formularul vizibil, dar pentru accesibilitate completă poți adăuga un fallback server-side (form cu `action`) care funcționează și fără JS.
- Managementul static files: în producție, nu folosi CDN direct în toate cazurile; gestionează static files cu `collectstatic` și un server static.

Contribuire

- Pentru modificări: clone repo, creează branch, rulează testele local, deschide PR.
- Respectă convențiile existente (stil, structura template-urilor)

Support / contact

Dacă observi vreun bug specific sau vrei să adaug funcționalități (ex: plată taxe, upload avatar, grafice), spune-mi aici și le implementez.

---

Acesta este un README complet care ar trebui să acopere toate aspectele importante ale proiectului. Pot extinde cu diagrame UML, exemple de requests cURL pentru API, sau un script de populare demo (fixtures) dacă dorești.
