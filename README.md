# TBO projekt
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
