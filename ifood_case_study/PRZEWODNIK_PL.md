# iFood Case Study: przewodnik po rozwiązaniu (PL)

Pełne rozwiązanie z kodem, wykresami i komentarzem po angielsku jest w `iFood_Case_Study_Solution.ipynb`. Ten plik to wersja do nauki i do obrony na rozmowie. Dla każdego zadania opisuję **podejście**, **co konkretnie zrobiłem**, **wynik** i **co ten wynik znaczy**. Na końcu jest słowniczek metryk i lista pytań, które mogą paść na rozmowie.

> Jednostki: kwoty w danych to „MU” (monetary units), nie wiadomo, w jakiej walucie. Wszystkie liczby poniżej pochodzą z danych **po czyszczeniu** (1 994 klientów), chyba że napisano inaczej.
>
> **Do prezentacji:** gotowy skrypt do przeczytania po angielsku (ok. 20 min, z ramkami „If they ask…”) jest w `PRESENTATION_SCRIPT_EN.md`.

---

## Najważniejsze w 7 punktach

1. **Dane są brudniejsze, niż mówi słownik.** Duplikaty stanowią ok. 9% wierszy, a 19 par klientów ma *sprzeczny* `Response`. `MntTotal` nie zawiera złota, `MntRegularProds` ma ujemne wartości, a `AcceptedCmpOverall` jest zdefiniowany inaczej, niż deklaruje słownik. Wiek i staż liczone są do daty z **2020** roku, choć dane pochodzą z lat 2012–14. Po czyszczeniu zostaje 2 205 → **1 994 klientów**.
2. **`Z_CostContact = 3` i `Z_Revenue = 11` to ekonomia kampanii.** Próg rentowności wynosi 3/11 = **27,3%** odpowiedzi. Kampania miała ok. 15% odpowiedzi, więc straciła **45%** swojego kosztu.
3. **Kto wydaje:** decyduje skład gospodarstwa domowego, a nie wykształcenie czy stan cywilny. **20% najlepszych klientów daje 52% przychodu.**
4. **Teza marketingu o dzieciach jest prawdziwa, ale mechanizm jest bardziej precyzyjny.** Przy tym samym dochodzie mniej wina kupują rodziny z *małymi dziećmi*. Na kampanie słabiej reagują rodziny z *nastolatkami*.
5. **Test reklamacji (Complain) nie ma mocy statystycznej.** Nawet gdyby żaden z 19 reklamujących nie odpowiedział, p wyniosłoby 0,096. Wynik jest **nierozstrzygnięty**, a nie „brak efektu”.
6. **Model:** wygrywa regresja logistyczna (PR-AUC 0,71 w nested CV, a na teście ROC-AUC 0,91). Kontaktowanie tylko klientów z p > 27% zamienia **−1 385 MU w +484 MU na 1 000 klientów**.
7. **Segmenty:** 4 persony. Żaden segment nie przekracza progu 27% w całości, dlatego **segment wyznacza komunikat, a model wyznacza, kogo kontaktować.**

---

## Decyzje: jakie były warianty, co wybrałem i dlaczego

W notatniku jest to „Decision log” na początku, a pełny opis każdej decyzji znajduje się w miejscu, w którym zapadła.

### 1. 19 klientów ze sprzecznym `Response` (Q1b)
| Wariant | Skutek | Ocena |
|---|---|---|
| **Jeden wiersz, etykieta „nieznana”** ✅ | 1 994 klientów w analizie opisowej, 1 975 w testach i modelu | **wybrane** |
| Usunąć klienta całkowicie | 1 975 wszędzie | tracimy poprawne dane o wydatkach i demografii, nic nie zyskując |
| Przyjąć `Response = 1` | odsetek odpowiedzi rośnie z 14,7% do ok. 15,5% | brak dowodu; zawyża KPI i uczy model być może błędnych etykiet |
| Zostawić obie kopie | ta sama osoba z dwiema przeciwnymi etykietami | szum; ta sama osoba może trafić do train i do test z przeciwnymi odpowiedziami |

**Dlaczego:** kopie to na pewno ten sam klient, ale w danych nie ma ID ani daty, które powiedziałyby, która etykieta jest prawdziwa. Brak etykiety to uczciwy zapis tego, co wiemy. Wydatki tych klientów są poprawne, więc zostają w EDA i segmentacji. To 1% etykiet, więc żaden wynik nie może się przez to istotnie zmienić.

### 2. 6 rekordów logicznie niemożliwych (Q1d)
| Wariant | Ocena |
|---|---|
| **Usunąć** ✅ | **wybrane** |
| Zostawić z flagą | skrajne wartości (28 zakupów z katalogu przy dochodzie 2 447) ciągnęłyby centroidy k-means i dopasowanie dochód–wydatki, a flaga nie mówi, *które* pole jest błędne |
| „Naprawić” wartości | czyste zgadywanie |

**Dlaczego:**
- Łamią **twarde reguły logiki**: wydatki przy zerze zakupów, więcej zakupów z rabatem niż wszystkich, wydatki = 35% dochodu. Nie chodzi o to, że wyglądają „dziwnie”.
- Mają wspólny, zepsuty wzorzec: dochód poniżej 10 tys., często 15 zakupów z rabatem albo 14–20 wizyt w sieci.
- To 0,3% bazy i sami nie-responderzy; odsetek odpowiedzi zmienia się tylko z 14,69% na 14,73%.

### 3. Podejrzane dochody: 23 poniżej 10 tys., w tym 12 × dokładnie 7 500 (Q1d, Q3c)
| Wariant | Ocena |
|---|---|
| **Zostawić i opisać** ✅ | **wybrane** |
| 7 500 → brak + imputacja | wymyśla dane i tworzy nowy „szpic” w miejscu imputowanej wartości |
| Usunąć dochody < 10 tys. | usuwa też prawdziwie biednych klientów, czyli przesuwa bazę w stronę zamożnych (selekcja, którą sami byśmy wprowadzili) |

**Dlaczego:** te wartości są *możliwe*, tylko podejrzane. Metody są na nie odporne (rangi, logarytm, regularyzacja). **Test odporności:** bez nich korelacja Spearmana zostaje 0,86, a stosunek wydatków rodzin do bezdzietnych ok. 0,63. Zmienia się tylko elastyczność (2,4 → 2,7), więc podaję ją jako zakres „ok. 2,4–2,7”.

### 4–5. Kolumny pochodne i liczba zakupów (Q1, Q2)
- **Przeliczam sumy z komponentów** zamiast im ufać, bo 2 z 3 są niezgodne ze słownikiem.
- `TotalPurchases` liczę zgodnie z poleceniem **i** w wersji bez podwójnego liczenia zakupów z rabatem. Do wartości koszyka używam wersji kanałowej (mediana 29,9 vs 23,7).

### 6. Model finalny (Q5): regresja logistyczna wybrana regułą 1-SE
**Dlaczego w ogóle te cztery modele (kandydaci):**
| Kandydat | Po co w wyścigu |
|---|---|
| Baseline (zawsze przewiduje średnią) | podłoga: każdy model musi go pobić (PR-AUC 0,147) |
| **Regresja logistyczna** (L1/L2/elastic net, z wagami klas i bez) | standard w modelach odpowiedzi: skalibrowane prawdopodobieństwa, współczynniki = ilorazy szans, dobra przy małych danych |
| Random forest | nieliniowości i interakcje bez ręcznego feature engineeringu |
| Gradient boosting (HistGB) | zwykle najdokładniejsza rodzina na danych tabelarycznych; odpowiednik XGBoost/LightGBM |

**Czego nie użyłem i dlaczego:**
- SVM: brak natywnych prawdopodobieństw.
- k-NN: słabe przy mieszanych cechach i niezbalansowanych klasach.
- Naive Bayes: zakłada niezależność cech, a wydatki są silnie skorelowane.
- Sieci neuronowe: 233 pozytywy to za mało.
- XGBoost: poza listą dozwolonych bibliotek.

**Porównanie (wszystkie modele na równych zasadach, każdy z własnym progiem z treningu):**
| Model | Nested CV PR-AUC | Test PR-AUC | Test ROC-AUC | Brier | Zysk test (własny próg) |
|---|---|---|---|---|---|
| **LR 1-SE** ✅ | 0,71 | 0,662 | 0,906 | 0,080 | +182 (przy 27,3%: **+191**) |
| LR najlepsze CV | 0,71 | 0,666 | 0,905 | 0,081 | +216 (próg 0,15) |
| Gradient boosting | 0,64 | 0,592 | 0,886 | 0,088 | +154 |
| Random forest | 0,60 | 0,508 | 0,874 | 0,094 | +151 |

**Czym charakteryzuje się wybrany model:**
- **Regresja logistyczna:** wynik klienta = ważona suma cech (log-szanse), a p = 1/(1+e^(−wynik)). Każda waga to iloraz szans, który da się wytłumaczyć CMO.
- **Elastic net (pół L1, pół L2):**
  - L1 zeruje słabe lub zbędne cechy (tu `Kidhome` i `AvgSpendPerOrder`), czyli wbudowana selekcja;
  - L2 rozkłada wagę między skorelowane cechy wydatków (VIF do 24), zamiast losowo wybierać jedną.
- **C = 0,32:** ok. 100× silniejsza regularizacja niż w wariancie „najlepsze CV” (C = 31,6). Jest potrzebna, bo na parametr przypada tylko **7,3 zdarzenia** (reguła kciuka dla modelu bez kary to ≥ 10).
- **Bez wag klas,** więc prawdopodobieństwa zachowują sens (kalibracja) i reguła „p > 27,3%” działa.
- **Reguła 1-SE:**
  - wynik CV sam jest szumny (SD między foldami 0,028), więc wszystko w odległości 1 błędu standardowego od najlepszego (AP ≥ 0,689) jest statystycznie nierozróżnialne;
  - z tej grupy biorę najprostszy model;
  - decyzję podjąłem **przed** otwarciem zbioru testowego.
- **Zmierzona stabilność** (50 prób bootstrapowych):
  - współczynniki są ok. **40% mniej zmienne** (średnie SD 0,155 vs 0,271);
  - mniejsze wagi (największa 1,15 vs 1,79);
  - taka sama dokładność;
  - zmiany znaku równie rzadkie w obu wariantach (ok. 7%), więc oba mówią tę samą historię, ale 1-SE robi to stabilniej.

**Z czego świadomie rezygnujemy (uczciwie):** wariant „najlepsze CV” z własnym progiem 0,15 zarobił więcej: na treningu (OOF) 977 vs 917, na teście +216 vs +191 MU. Różnica +25 MU ma jednak 95% CI od −30 do +87 (nieistotna; w 20% losowań wypada gorzej). Wybieram stabilność, prostotę i przejrzystą regułę zamiast zysku, który może być szumem. Gdyby priorytetem był maksymalny oczekiwany zysk, ten wariant jest opisaną alternatywą (z walidacją progu przed każdą kampanią).

### 7. Zestaw cech (Q5)
| Wariant | CV PR-AUC | Jakie ryzyko adresuje |
|---|---|---|
| **Wszystkie cechy** ✅ | **0,694** | – |
| Bez Recency | 0,621 | wyciek czasowy, jeśli recency mierzono po kampanii |
| Bez demografii (wiek, dzieci, stan cywilny, wykształcenie) | 0,646 | bezpośrednie użycie cech osobowych |
| Bez Recency i demografii | 0,578 | oba ryzyka naraz |
| Bez historii kampanii | 0,547 | klienci bez historii („cold start”) |

**Dlaczego wszystkie:**
- Najwyższa dokładność.
- **Recency to pytanie o pochodzenie danych, a nie o model.** Trzeba potwierdzić datę snapshotu; wariant bez Recency czeka jako zapas (0,62, nadal 4× lepiej niż losowo).
- **Usunięcie demografii nie usuwa uprzedzeń,** bo dochód i koszyk działają jak zastępniki, a do tego trudniej wtedy mierzyć uprzedzenia. Lepiej zostawić te cechy, **mierzyć wyniki w grupach** i wprowadzić zasady (np. minimalny dostęp do oferty).

### 8. Kogo kontaktować (Q5): próg rentowności 27,3%
| Reguła | Kontakt | Wychwyceni responderzy | Zysk test | Na 1 000 klientów | Kiedy ma sens |
|---|---|---|---|---|---|
| Wszyscy | 100% | 100% | −547 | −1 385 | nigdy przy tej ekonomii |
| Reguła kciuka („przyjął wcześniej”) | 22% | 52% | +75 | +190 | brak modelu |
| Próg 0,5 | 9% | 43% | +167 | +423 | mały budżet / zmęczenie klientów kontaktami |
| Próg z danych (OOF) 0,26 | 19% | 64% | +182 | +461 | próg czysto empiryczny |
| **Próg rentowności 27,3%** ✅ | **18%** | **64%** | **+191** | **+484** | **domyślnie** |
| Stały budżet: top 20% | 20% | 66% | +181 | +458 | budżet ustalony z góry |
| *Idealna wiedza* | 15% | 100% | +464 | +1 175 | teoretyczny sufit |

**Dlaczego próg rentowności:**
- Wynika z ekonomii, a nie z dopasowania do danych (11·p > 3), więc nie może się przeuczyć.
- Jest poprawny, bo model jest skalibrowany: próg z danych (0,26) wychodzi praktycznie taki sam.
- Dał najwyższy zysk spośród reguł, które da się ustalić z góry.
- Sam się aktualizuje: próg = koszt / wartość odpowiedzi; przy marży 40% byłoby to 3/4,4 ≈ 68%.

**Inne możliwości:**
- stały budżet, gdy kwota jest ustalona wcześniej;
- niższy próg w kampaniach zasięgowych;
- wyższy, gdy problemem jest zmęczenie klientów kontaktami;
- docelowo próg „uplift” (kontakt tylko z osobami, które odpowiedzą *dzięki* ofercie).

### 9–10. Liczba segmentów i skalowanie (Q6): k-means, k = 4, log + standaryzacja
| Wariant | Silhouette | Stabilność (ARI) | Najmniejszy segment | Znaczenie | Ocena |
|---|---|---|---|---|---|
| k = 2 | 0,38 (najwyższa) | 0,97 | 47% | tylko „lekcy vs ciężcy kupujący” | za mało do działania |
| k = 3 | 0,31 | 0,98 | 29% | łączy rodziców nastolatków szukających promocji z zamożnymi rodzinami | ukrywa dwie różne grupy |
| **k = 4** ✅ | 0,25 | **0,87** | **20%** | 4 persony, każda z inną akcją | **wybrane** |
| k = 5 | 0,19 | 0,91 | 18% | dzieli niskie wydatki na 41 i 105 MU przy tej samej demografii | brak nowej akcji |

**Dlaczego k = 4:**
- Dane to raczej kontinuum niż wyraźne „wyspy” (silhouette spada od k = 2), więc k jest częściowo decyzją biznesową w granicach statystyki.
- k = 4 jest stabilne, a każdy segment ma co najmniej 20% klientów.
- **Każdy segment wymaga innej akcji.**

**Skalowanie:** log + z-score sprawia, że 10→100 znaczy tyle samo co 100→1000, a kilku dużych klientów nie dominuje odległości.

---

## Q1: Jakość danych

**Podejście.** Traktuję to jako audyt, a nie formalność. Usuwam tylko to, co jest **dowodliwie błędne**. Wartości skrajne, ale możliwe, zostają i obsługuję je później metodami odpornymi (rangi, logarytm). Każdy krok czyszczenia zapisuję z liczbą wierszy przed i po.

**Co zrobiłem i co wyszło:**

| Krok | Co zrobiłem | Wynik | Co to znaczy |
|---|---|---|---|
| a) Struktura | shape, typy, braki, liczba unikalnych wartości | 2 205 × 39, **0 braków**; `Income` jest typu float, ale wszystkie wartości są całkowite | float to „ślad” po brakach. Klientów bez dochodu najpewniej **usunięto po cichu** (selekcja do potwierdzenia u właściciela danych) |
| b) Duplikaty dokładne | `df.duplicated()` | **184 (8,3%)**: 172 klientów ×2 i 6 klientów ×3 | zawyżają liczebność (zbyt pewne p-value). W modelu kopie trafiają do train i test, czyli dają **wyciek** |
| b) Duplikaty ze sprzecznym targetem | duplikaty na 38 kolumnach bez `Response` | **19 par**: ten sam klient raz z `Response`=1, raz z 0 | nie wiadomo, która etykieta jest prawdziwa, więc zostawiam 1 wiersz z `Response` = **NaN** (dane o wydatkach zostają w analizie opisowej, ale nie biorą udziału w testach i modelu) |
| b) Prawie-duplikaty | duplikaty na wszystkim poza jednym atrybutem | **2 pary** różnią się tylko stanem cywilnym | ta sama osoba, zostawiam jeden rekord |
| b) Dowód, że to kopie | czy różni klienci mogą mieć tę samą parę (Income, Customer_Days)? | tylko 1 para „prawdziwie różnych” klientów, i ta różni się w 18 kolumnach | zgodność na 36–38 kolumnach to kopia, a nie przypadek |
| c) `Z_*` | nunique, std | stałe: **3** i **11** | jako cechy są bezużyteczne, ale to koszt kontaktu i przychód z odpowiedzi (zgadza się co do jednostki z publicznym briefem iFood: 2 240×3 = 6 720 i 334×11 = 3 674). **Próg rentowności 27,3%.** Usuwam je jako kolumny, zostawiam jako stałe |
| extra: kolumny pochodne | przeliczenie z komponentów | `MntTotal` = 5 kategorii **bez złota** (100% wierszy); `MntRegularProds` odejmuje złoto drugi raz, więc ma 3 wartości ujemne (min −283); `AcceptedCmpOverall` = tylko kampanie 1–5 (**bez** `Response`) | przychód był zaniżony o 7,8%. `AcceptedCmpOverall` nie przecieka, ale ktoś „naprawiający” go zgodnie ze słownikiem wprowadziłby wyciek targetu. Dlatego przeliczam wszystkie trzy kolumny sam |
| d) Wiek | zakres + data odniesienia | 24–80 lat, wartości możliwe. **Ale** `Customer_Days` obejmuje dokładnie 699 dni, czyli tyle, ile okno zapisów w danych źródłowych (30.07.2012 – 29.06.2014). Data odniesienia to więc **27.05.2020** | wiek i staż są zawyżone o ok. 6 lat (w czasie kampanii klienci mieli ok. 18–74 lat). Porównań to nie psuje (stałe przesunięcie), ale **wartości bezwzględne są błędne** |
| d) Dochód | Tukey, kwantyle, heaping, relacja do wydatków | brak outlierów wg Tukeya. **12 osób ma dokładnie 7 500** (placeholder), 29 osób ma < 10 tys., a jedna osoba wydała 35% dwuletniego dochodu (99. percentyl to 1,4%) | problem to placeholdery i niewiarygodne wartości, a nie „klasyczne” outliery |
| d) Recency | test χ² zgodności z rozkładem jednostajnym | 0–99 dni, rozkład **idealnie płaski** (p = 0,97) | prawdziwa recency jest prawostronnie skośna. Wyciąg jest więc przycięty lub przefiltrowany: brak klientów nieaktywnych (o churnie nic nie powiemy) i nie wiadomo, *kiedy* recency mierzono (ryzyko wycieku w Q5) |
| d) Reguły logiczne | wydatki przy 0 zakupów; więcej zakupów z rabatem niż wszystkich zakupów; wydatki > 25% dochodu | **6 rekordów**, wszystkie z dochodem < 10 tys. i 14–20 wizytami w sieci miesięcznie (wyglądają na boty lub zepsute dane) | usuwam je (0,3%, sami „nie-responderzy”) |

**Wynik końcowy:** 2 205 → **1 994 klientów**, w tym 1 975 z wiarygodną etykietą i 291 responderów (**14,73%**).

**Q1e, co powiedzieć marketingowi (4 punkty):** ok. 1 na 10 rekordów to duplikat; wyliczone wcześniej sumy są błędne; daty liczone są do 2020 roku, a recency jest sztucznie płaska; dochód zawiera placeholdery. Do tego jedna liczba powinna być na każdym dashboardzie kampanii: **próg rentowności 27%**. „Ładnie wyglądające” 15% to w rzeczywistości strata.

---

## Q2: Feature engineering

**Podejście.** Każda cecha ma odpowiadać na pytanie biznesowe i jest **walidowana**. Jeśli definicja z polecenia ma wadę, robię ją zgodnie z poleceniem **i** dodaję poprawioną wersję.

| Cecha | Co zrobiłem | Wynik i znaczenie |
|---|---|---|
| `TotalChildren` | Kidhome + Teenhome (oraz `HasChildren` 0/1) | „rodzina” w języku marketingu; w Q3 okazuje się najsilniejszym czynnikiem różnicującym wydatki |
| `TotalPurchases` | zgodnie z poleceniem: deals + web + catalog + store | **Wada:** zakup z rabatem i tak odbywa się w jakimś kanale, więc deals są liczone podwójnie. Dodałem `ChannelPurchases` (bez deals). Mediana 15 vs 12 |
| `AvgSpendPerPurchase` | MntTotal (poprawiony, ze złotem) / TotalPurchases; przy 0 zakupów **NaN** (nie 0 i nie ∞) | mediana 23,7 vs **29,9** dla `AvgSpendPerOrder` (bez podwójnego liczenia), czyli wersja z polecenia zaniża koszyk o ok. 20%. Ważne dla Q4c (próg 30 $ leży dokładnie na medianie koszyka) |
| `TenureYears` | staż **względem najnowszego klienta** (0–1,9 roku) + wersja „jak w danych” (5,9–7,8) + odtworzona data zapisu | daty zapisu wychodzą dokładnie 2012-07-30 → 2014-06-29, co potwierdza wniosek z Q1d |
| `Education`, `Marital_Status` | zwinięcie dummies do jednej kategorii (`idxmax`) + test „w obie strony” | grupy Basic (49) i Widow (69) są małe, więc w dalszej analizie dostają szerokie przedziały ufności. „2n Cycle” to bolońskie studia II stopnia (≈ magister), więc kolejność jest niejednoznaczna i nie łączę kategorii na siłę |

---

## Q3: EDA i storytelling

**Podejście.** Średnia *i* mediana, skośność; wydatki również w skali logarytmicznej. Segmenty porównuję przez **udział w przychodzie vs udział w klientach** (indeks), a który wymiar różnicuje najbardziej, sprawdzam testem **Kruskala-Wallisa** z wielkością efektu η² (przy ~2 000 obserwacji prawie wszystko jest „istotne”, więc liczy się efekt). Miks wydatków liczę jako udział w portfelu ważony przychodem.

**Wyniki:**
- **a) Rozkłady:**
  - dochód jest symetryczny (średnia 51,8 tys. ≈ mediana 51,5 tys.);
  - wiek jest symetryczny (ale liczony do 2020 roku);
  - **wydatki są mocno prawoskośne** (średnia 608 vs mediana 398);
  - w skali log rozkład jest **dwumodalny** (mieszanina 2 rozkładów: BIC 3 213 vs 3 896): ok. 42% lekkich kupujących (~60 MU) i ok. 58% ciężkich (~850 MU).

  **Znaczenie:** „przeciętny klient” nie istnieje. Raportuj mediany i segmenty, a w modelach używaj logarytmu.
- **b) Segmenty:**
  - **Dzieci różnicują najmocniej** (η² = 0,26), wykształcenie słabo (0,03), stan cywilny wcale (0,00, p = 0,22).
  - Bezdzietni to **28% klientów i 52% przychodu**.
  - Basic (49 osób) to zupełnie inny koszyk: wino stanowi ~9% (indeks 18), a owoce, ryby, słodycze i złoto mają indeks ponad 300.
  - PhD to wino (indeks 121).
  - Bezdzietni kupują mięso i ryby (indeksy 122/111). Rodziny wydają *większą część mniejszego portfela* na wino (112–123) i złoto (119–144).
- **c) Dochód a wydatki:**
  - korelacja Spearmana 0,86;
  - **elastyczność 2,4**, czyli +10% dochodu daje +25% wydatków (zależność wypukła, dobro „luksusowe”);
  - udział wydatków w dochodzie rośnie z 0,11% do 0,86% między kwartylami dochodu;
  - **przy tym samym dochodzie rodziny wydają dużo mniej** (np. w 3. decylu 66 vs 290 MU).
- **d) Trzy slajdy dla szefa marketingu:**
  1. **Pareto:** 20% klientów daje 52% przychodu, a dolna połowa tylko 9%. → Chroń i rozwijaj górne 20% (VIP, retencja) zamiast dawać rabaty wszystkim.
  2. **Luka „dzieci”:** przy tym samym dochodzie rodziny wydają o 20–72% mniej. → Targetowanie po samym dochodzie przeszacowuje rodziny; dla rodzin sprawdzą się zestawy i budowanie koszyka.
  3. **Historia kampanii a próg rentowności:** 79% klientów nigdy nic nie przyjęło i odpowiada w 8%; ci, którzy przyjęli wcześniej, odpowiadają w 30–82%. → Koniec z masową wysyłką.

---

## Q4: Testy statystyczne

**Podejście.**
- **Wino:** dane są skośne i mają dużo zer (Shapiro-Wilk p < 10⁻¹¹), więc test główny to **Mann-Whitney U**. Dla odporności dodaję Welcha na średnich (średnia × liczba klientów = przychód) i na log.
- **Response:** zmienna 0/1, więc **χ²** (równoważny testowi z dla dwóch proporcji) i **Fisher** przy małych liczebnościach.
- **Wielkości efektu:**
  - wino: różnica średnich i median z **bootstrapowym 95% CI**, **delta Cliffa**;
  - response: **różnica ryzyk z CI Newcombe'a**, RR, OR, h Cohena.
- **Korekta Holma** na wielokrotne testowanie.
- **Confounding:** rodziny zarabiają mniej, więc kontroluję dochód i wiek (OLS z odpornymi błędami HC1 i regresja logistyczna, dochód na dwa sposoby), rozdzielam małe dzieci od nastolatków i dodaję test **Cochrana-Mantela-Haenszela** w kwartylach dochodu.

**Q4a, wynik:**
- **Wino:** średnia 232 vs 500 MU (**−54%**, CI różnicy −301…−235), mediana 97 vs 466, **delta Cliffa −0,44** (efekt średni do dużego; losowy bezdzietny wydaje więcej od losowego rodzica w 72% przypadków), p ≈ 10⁻⁵¹.
- **Response:** **9,6% vs 27,8%** (−18 p.p., CI −22,4…−14,3), RR 0,34 (rodzice odpowiadają ~3× rzadziej). „Jakakolwiek z 6 kampanii”: 19,8% vs 45,8%. Holm: wszystko istotne.
- **Ale przy tym samym dochodzie:**
  - na wino wpływają **małe dzieci** (−38% do −52%), a **nastolatki nie** (od 0 do +19%);
  - na response wpływają **nastolatki** (OR 0,34–0,41), a **małe dzieci nie** (OR ≈ 0,9, nieistotne);
  - CMH: test związku p ≈ 3·10⁻¹⁰, ale **efekt nie jest jednorodny** (test interakcji p = 0,007). W najniższym kwartylu dochodu wszyscy odpowiadają w ok. 10% (OR 1,08), w pozostałych efekt jest silny (OR 0,22–0,38). Wspólny OR 0,35 to tylko średnia, dlatego raportuję wyniki osobno dla każdego przedziału dochodu.
- **Znaczenie:** marketing ma rację co do obu tez, ale etykieta „ma dzieci” jest za grubym narzędziem. Rodziny z małymi dziećmi i rodziny z nastolatkami to dwie różne grupy odbiorców. Zastrzeżenie: to dane obserwacyjne. Mniejsza reakcja może oznaczać „mniejszą reakcję na *nasze* oferty”, a to rozstrzygnie dopiero eksperyment z ofertą rodzinną.

**Q4b, Complain:**
- 3/19 = 15,8% (CI 5,5–37,6%) vs 14,7%; różnica +1,1 p.p. (CI **−9,3…+22,9**); Fisher p = 0,75.
- **Czy wynik jest wiarygodny? Nie, jest nierozstrzygnięty.**
  - Nawet gdyby **0 z 19** odpowiedziało, p = **0,096**, więc moc testu dwustronnego dla *jakiegokolwiek spadku* wynosi **0%**. Przybliżenie normalne „obiecuje” do 93% mocy, bo przy ~3 oczekiwanych zdarzeniach przestaje działać.
  - Do wykrycia **spadku o połowę** potrzeba ok. **149 reklamujących** (mamy 19), co przy 1% reklamacji oznacza bazę ok. 15,6 tys. klientów.
  - Bayes: P(reklamujący odpowiadają rzadziej) = 34%.
- **Rada dla interesariusza:** z tych danych nie da się uzyskać definitywnej odpowiedzi. Zbierajmy dane w kolejnych kampaniach (ok. 150 osób), wykorzystajmy wszystkie 6 kampanii (model mieszany / GEE), doprecyzujmy pytanie (reklamacja przed czy po ofercie, rozwiązana czy nie). Do tego czasu przyjmijmy politykę „najpierw obsługa reklamacji, potem promocja”.

**Q4c, projekt testu A/B (darmowa dostawa od 30 $):**
- **H0/H1:** H0: brak różnicy w **marży na klienta**; test dwustronny (oferta może też tracić pieniądze).
- **Jednostka randomizacji:** klient (konto), a nie sesja czy zamówienie. Przydział przez hash ID + sól, stratyfikacja (segment, miasto, aktywność), 50/50. Przed startem test A/A i kontrola SRM.
- **Metryka główna:** marża wkładu na klienta (obejmuje i wzrost popytu, i koszt dopłaty do dostawy).
- **Guardraile:** koszt dostawy na zamówienie, **czas dostawy / SLA**, anulacje, reklamacje, wypisy, „dopychanie koszyka” do 30–32 $, retencja po teście.
- **Liczebność i czas trwania:** na danych: miesięczne wydatki 25,3 ± 25,1, więc dla +5% potrzeba **~6 100 klientów na wariant**, a z CUPED ok. 3 100. Minimum **2 pełne tygodnie**, bez podglądania wyników. **50% klientów i tak ma koszyk ≥ 30**, więc dostaną darmową dostawę bez zmiany zachowania (czysty koszt).
- **Co może unieważnić wynik:**
  - **interferencja** przez wspólną flotę kurierów (naruszenie SUTVA, bo kontrola też czeka dłużej);
  - **kontaminacja** (wspólne konta, kody krążące w sieci);
  - efekt nowości i przesunięcie zamówień w czasie;
  - sezonowość;
  - błędy wykonania (SRM, podglądanie wyników).

---

## Q5: Model predykcyjny

**Podejście.**
- **Audyt wycieku.** `AcceptedCmpOverall` sprawdziłem w Q1: nie zawiera targetu, ale jest sumą Cmp1–5, więc go pomijam i zostawiam 5 flag. Pomijam też sumy i kombinacje liniowe, stałe `Z_*` oraz 19 rekordów ze sprzeczną etykietą.
- **Podział:** stratyfikowany 80/20; zbiór testowy używany **raz**.
- **Pipeline:** imputacja, log1p, skalowanie i one-hot **w środku pipeline'u** (bez wycieku z preprocessingu). Kategorie referencyjne to Graduation i Married.
- **Kilka modeli:** dummy, regresja logistyczna (L1/L2/elastic net × wagi klas, **grid search**), random forest i gradient boosting (**random search**).
- **Kilka sposobów walidacji:**
  - 5-fold CV do strojenia;
  - **nested CV** (5×3) do uczciwego porównania;
  - porównanie parami na tych samych foldach;
  - repeated CV (5×3) dla wariantów;
  - holdout z **bootstrapowymi CI**;
  - **reguła 1-SE**: najprostszy model statystycznie nie gorszy od najlepszego.
- **Metryki:** PR-AUC (główna), ROC-AUC, Brier, lift oraz **zysk z kampanii** liczony z ekonomii 3/11. Próg wybieram na predykcjach **out-of-fold z treningu**, nigdy na teście.

**Wyniki:**
- **Wybór modelu (przed testem):**
  - nested CV PR-AUC: **LR 0,71**, GB 0,64, RF 0,60 (baza 0,15). LR wygrywa na 5/5 foldach.
  - Finalny model: **LR wybrana regułą 1-SE** (C = 0,32, elastic net, bez wag klas). Jest prostsza, stabilniejsza i dobrze skalibrowana.
  - Dlaczego prosty model wygrywa: mało danych (233 pozytywy w treningu) i głównie monotoniczne zależności, a drzewa się przeuczały.
- **Dlaczego accuracy jest złą metryką:** „nikt nie odpowie” daje **85,3% accuracy** i zero znalezionych klientów. Model przy progu 0,5 ma 88,9%. Różnica 3,5 p.p. ukrywa całą wartość modelu, a do tego accuracy traktuje stratę 3 MU (zbędny kontakt) tak samo jak 8 MU (utracony zysk).
- **Test (395 klientów, 58 responderów):** PR-AUC **0,66** (CI 0,55–0,76), ROC-AUC **0,91**, Brier 0,080 (baza 0,125), **lift 4,4×** w top 10%, kalibracja dobra (przewidziane 0,26 → zaobserwowane 0,29). Zgadza się z nested CV, więc model nie jest przeuczony.
- **Biznes:**
  - próg teoretyczny 11p > 3, czyli **p > 27,3%**; krzywa zysku OOF niezależnie daje szczyt przy **0,26** (potwierdzenie kalibracji);
  - na teście: kontakt do wszystkich daje **−547 MU** (−1 385 na 1 000 klientów); reguła kciuka „przyjął wcześniej” daje +75; **model daje +191 MU (+484 na 1 000)**, kontaktując 18% klientów i docierając do 64% responderów;
  - top 20% wg modelu zawiera **66% wszystkich responderów**.
- **Top 3 czynniki** (dwa różne algorytmy zgodne w 96% rang):
  1. **Historia kampanii:** 0 wcześniejszych akceptacji → 8%, 1 → 30%, 2 → 52%, 3+ → 82%.
  2. **Recency:** zakup w ostatnich 0–19 dniach → 29% vs 80–99 dni → 6%.
  3. **Kanał:** 6+ zakupów z katalogu → 31% vs 0 → 4%.

  Dalej: staż (najdłuższy kwartyl 26% vs najnowszy 9%), nastolatki (−) i stan cywilny (single/rozwiedzeni/wdowcy ~2–3× więcej niż w związku). Dochód, wiek i reklamacje nic nie wnoszą, gdy znamy zachowanie.

  **Rekomendacja:** „Kontaktuj tych, którzy już kiedyś przyjęli ofertę, kupili w ostatnich ~3 tygodniach i kupują z katalogu; tylko tych z wynikiem modelu > 27%”.
- **Odporność (AP):** wszystkie cechy 0,694; **bez Recency 0,621** (gdyby recency mierzono *po* kampanii, byłby to wyciek czasowy, więc trzeba to potwierdzić); bez demografii 0,646; bez historii kampanii 0,547.
- **Q5d, ryzyka (zmierzone):**
  - Model wybiera **34% bezdzietnych vs 8–13% rodzin** oraz 34% najbogatszego kwartyla vs 11–13% pozostałych.
  - Wśród responderów wychwytuje 79% bezdzietnych, ale tylko 50% rodzin z 2+ dzieci.
  - Rabaty trafiłyby więc do najbogatszych, którzy i tak kupują: to **regresywność**, ryzyko wizerunkowe i ryzyko **proxy-dyskryminacji**.
  - Kolejne ryzyka:
    - **response ≠ inkrementalność** (potrzebny holdout i model uplift);
    - pętla sprzężenia zwrotnego (zostaw 5–10% losowych kontaktów);
    - 11 MU to przychód, a nie marża, i nie obejmuje kosztu rabatu (realny próg jest wyższy);
    - RODO (profilowanie, przejrzystość, art. 21/22);
    - dryf danych i mała liczba pozytywów.

---

## Q6: Segmentacja

**Podejście.**
- **Cechy:** tylko zachowanie (6 kategorii wydatków, 4 kanały zakupów, wizyty w sieci). Demografię pomijam celowo, żeby później nią **walidować** segmenty. Recency pomijam, bo jest płaska i zmienia się codziennie.
- **3 warianty skalowania:** z-score, **log1p + z-score** (wybrany) i rangi. Silhouette nie da się porównać między różnymi przestrzeniami cech, więc wybór jest koncepcyjny: logarytm traktuje 10→100 tak samo jak 100→1000 i nie pozwala ogonowi najwięcej wydających zdominować odległości.
- **Wybór k:** łokieć, silhouette, Calinski-Harabasz, Davies-Bouldin, **stabilność bootstrap (ARI)** i dwa alternatywne algorytmy (GMM z BIC, Ward).

**Wynik: k = 4.**
- Silhouette jest najwyższe przy k = 2, ale k = 2 to tylko „lekcy vs ciężcy kupujący”.
- k = 4 jest stabilne (ARI 0,87), najmniejszy segment ma 20%, a Ward zgadza się z k-means (ARI 0,71).
- k = 3 zlewa dwa różne segmenty.
- k = 5 dzieli tylko grupę o niskich wydatkach na „bardzo niskie” i „niskie”, co nie daje innej akcji.

| Persona | Udział / przychód | Profil | Akcja |
|---|---|---|---|
| 0 **Budget Young Families** | 36% / **4%** | dochód ~32 tys., 77% ma małe dzieci, 60 MU w 2 lata, 34% zakupów z rabatem, często przeglądają stronę, odpowiedź 7% | **bez płatnego kontaktu** (−2,2 MU na kontakt); darmowe kanały własne, zestawy rodzinne |
| 1 **Deal-Seeking Teen Parents** | 21% / 16% | ~49 tys., 75% ma nastolatki, **wino ~70% koszyka**, najwięcej zakupów z rabatem, kanał web, odpowiedź 16% | promocje winne online (pakiety, klub), kontakt tylko do osób wybranych przez model |
| 2 **Established Family Foodies** | 20% / 34% | ~63 tys., nastolatki, ~1 000 MU we wszystkich kategoriach, omnichannel, odpowiedź 16% | cross-sell świeżych i premium produktów, punkty lojalnościowe zamiast rabatów |
| 3 **Affluent Childless Connoisseurs** | 22% / **47%** | ~76 tys., 84% bez dzieci, 1 300 MU, mięso + wino, **katalog**, prawie bez rabatów, 39% przyjęło wcześniej, odpowiedź 25% | **program VIP / retencja**: premium, wczesny dostęp, bez rabatów |

**Kluczowy wniosek:** nawet najlepszy segment (25%) jest **poniżej progu 27,3%**, więc żadnego segmentu nie opłaca się kontaktować w całości. **Segment = komunikat i kanał, model = kogo kontaktować.**

---

## Q7: SQL i raport cykliczny

**Podejście.** Podaję zapytanie „podręcznikowe” i „produkcyjne”. Produkcyjne usuwa duplikaty (`DISTINCT`), przelicza `MntTotal` razem ze złotem i buduje `Education` z flag; nazwę `"education_2n Cycle"` trzeba wziąć w cudzysłów (zawiera spację), a `CAST` chroni przed całkowitoliczbowym `AVG` (SQL Server). Oba zapytania **uruchamiam** w SQLite i sprawdzam asercją, że wersja produkcyjna daje **dokładnie** to samo co pandas.

**Wynik:** po naprawie danych **zmienia się ranking**. Na surowej tabeli Master (571,0) jest przed Graduation (570,9), a po naprawie Graduation (624,0) wyprzedza Master (619,9). Średnie rosną o 4% (PhD) do 42% (Basic, którego portfel opiera się na złocie). Raport zbudowany na surowej tabeli pokazywałby złą kolejność.

**Q7b, pipeline:**
1. **Ingestion:** ELT/CDC do warstwy raw; zdarzenia kampanii jako eventy z czasem (to likwiduje problem timingu recency).
2. **Transformacje (dbt):** staging → intermediate → marts. Sumy liczone zawsze z komponentów, wiek i staż względem daty snapshotu; definicje metryk w warstwie semantycznej.
3. **Testy jakości jako bramka**, która przerywa uruchomienie. Każdy problem z Q1 staje się testem:
   - unikalność ID;
   - jedna odpowiedź na klienta i kampanię;
   - `MntTotal` = suma kategorii, `MntRegularProds` ≥ 0;
   - wariancja > 0;
   - dokładnie jedna flaga dummy;
   - zakresy wieku i dochodu, heaping;
   - zakupy z rabatem ≤ wszystkie zakupy;
   - świeżość danych i dryf (PSI).
4. **Orkiestracja:** Airflow / Dagster / dbt Cloud raz w tygodniu, alerty na Slacku, „dane na dzień X” plus status ostatniego testu jakości na dashboardzie.
5. **Dashboard:** KPI **vs próg 27%**, zysk kampanii, segmenty, panel jakości danych.
6. **Monitoring modelu:** PR-AUC, kalibracja, PSI i losowy holdout.

---

## Q8: Podsumowanie dla CMO (treść w notatniku, 190 słów)

Cztery zdania kluczowe: kampania do wszystkich traci pieniądze (próg 27%, wynik 15%). Wartość jest skoncentrowana (20% klientów → 52% przychodu). Przeszłe zachowanie przewiduje odpowiedź (30–82% vs 8%). **Rekomendacje:** koniec z masową wysyłką i kontakt tylko według modelu (−1 385 → +480 na 1 000 klientów), program VIP dla zamożnych bezdzietnych (22% klientów, 47% przychodu), naprawa danych (co 10. rekord to duplikat).

---

## Dlaczego wolno użyć każdej metody i dlaczego akurat tej

Każda metoda ma założenia, a wynik jest tyle wart, ile one. W notatniku (Dodatek A) **sprawdzam założenia na tych danych kodem**. Tu jest skrót.

**Sprawdzenia (wynik z notatnika):**
| Założenie | Wynik | Wniosek |
|---|---|---|
| Niezależność obserwacji (1 wiersz = 1 klient) | 0 duplikatów po czyszczeniu | ✅ testy, bootstrap i CV są uprawnione |
| χ²: oczekiwana liczebność ≥ 5 w każdej komórce | dzieci × response: 82; reklamacje × response: **2,8** | ✅ dla dzieci; ❌ dla reklamacji, dlatego tam Fisher |
| CMH: podobny OR w każdym przedziale dochodu | test interakcji p = 0,007 (OR 1,08 / 0,32 / 0,38 / 0,22) | ❌, więc podaję wyniki per przedział, a nie jeden OR |
| Regresja logistyczna (Q4): ≥ 10 zdarzeń na parametr | 26–97 | ✅ |
| Model Q5: ≥ 10 zdarzeń na parametr | **7,3** (233 / 32) | ⚠️ dlatego model jest **regularyzowany** |
| Współliniowość (VIF) w Q5 | do 24 (wartość koszyka, wino, mięso) | ⚠️ dlatego L2 w elastic net; L1 wyzerował wartość koszyka |
| Współliniowość kontroli w Q4 | VIF ≤ 1,3 | ✅ |
| Liniowość log-szans w Q5 | model ze splajnami 0,698 vs liniowy 0,702 | ✅ forma liniowa wystarcza |
| k-means: zwarte, kuliste klastry | zgodność z GMM (ARI 0,59) i Ward (0,71) | ⚠️/✅ wspólna struktura, granice nieco różne (brak ostrych klastrów) |
| Kalibracja (potrzebna do progu 27,3%) | Brier 0,080 vs 0,125 dla bazy; przewidziane 0,26 → zaobserwowane 0,29 | ✅ |

**Testy statystyczne: dlaczego wolno i dlaczego te:**
| Metoda | Założenia | Dlaczego tu można / dlaczego ta |
|---|---|---|
| Kruskal-Wallis + η² | niezależne obserwacje, wynik co najmniej porządkowy | wydatki są skośne (ANOVA zakłada normalność); η², bo przy 2 000 osób „istotne” jest prawie wszystko |
| Mann-Whitney + delta Cliffa | niezależne grupy; przy różnych kształtach rozkładów testuje P(X > Y), a nie samą medianę | dlatego wynik podaję jako „72% szans, że bezdzietny wydaje więcej”; nie wymaga normalności |
| Welch t | średnie z prób w przybliżeniu normalne (CTG przy n = 560 / 1 434), różne wariancje dozwolone | biznes liczy średnie (przychód); Welch, bo wariancje są różne |
| Bootstrap | obserwacje niezależne i z tego samego rozkładu | daje CI dla mediany, AP i zysku, które nie mają prostego wzoru |
| χ² / test z dla proporcji | liczebność oczekiwana ≥ 5 | spełnione dla dzieci |
| Fisher dokładny | brak wymagań co do liczebności | użyty tam, gdzie χ² nie działa (reklamacje) |
| Wilson / Newcombe | dane dwumianowe | klasyczny przedział Walda zawodzi przy małym n |
| OLS + błędy HC1 | liniowość parametrów, brak idealnej współliniowości; HC1 znosi wymóg stałej wariancji; normalność błędów niepotrzebna przy n ≈ 2 000 | log1p dla skośnych wydatków; ten sam wniosek przy elastycznej kontroli dochodu (decyle) |
| Logistyczna (inferencja) | niezależność, liniowe log-szanse, ≥ 10 zdarzeń na parametr | naturalny model dla 0/1; daje skorygowane OR |
| Holm | brak (działa przy dowolnej zależności testów) | ta sama ochrona co Bonferroni, a nigdy mniejsza moc |
| Moc: przybliżenie + symulacja | przybliżenie wymaga dużych liczebności | przy 2,8 oczekiwanych zdarzeniach przybliżenie kłamie (93%), symulacja dokładna pokazuje 0% |
| Bayes Beta-dwumianowy | dane dwumianowe, płaski prior | daje zdania typu „34% szans, że reklamujący odpowiadają rzadziej” |

**Modele ML: dlaczego wolno:**
| Model | Wymagania / założenia | Stan na tych danych |
|---|---|---|
| Regresja logistyczna | liniowe log-szanse, niezależność, dość zdarzeń na parametr, umiarkowana współliniowość | liniowość ✅ (splajny nic nie dają); mało zdarzeń i wysoki VIF ⚠️ obsłużone regularizacją ✅; kalibracja ✅ |
| Random forest | brak założeń rozkładowych; niezależne wiersze; prawdopodobieństwa nieskalibrowane | ✅ dozwolony; w praktyce przeucza się (233 pozytywy) i ma gorszą kalibrację |
| Gradient boosting | jak wyżej; wymaga regularizacji przy małych danych | ✅ dostroiłem learning rate, wielkość liścia i L2 |
| Stratyfikowana CV / nested CV | wiersze wymienne (brak porządku czasowego) | ✅ dla tego snapshotu. ⚠️ W produkcji kampanie są sekwencyjne, więc walidacja czasowa (ucz na przeszłych, testuj na następnej) |
| Grupowa permutation importance | skorelowane cechy zaniżają ważność pojedynczych kolumn | ✅ tasuję całe grupy naraz |

**Klastrowanie: dlaczego wolno:**
| Metoda | Założenia | Stan |
|---|---|---|
| k-means | sensowna odległość euklidesowa (cechy w tej samej skali), zwarte i podobnie rozłożone klastry, wrażliwość na outliery | ✅ log + z-score i usunięte niemożliwe rekordy; zgodność z GMM i Ward |
| GMM, Ward | (sprawdzenie) | rozluźniają założenia k-means, więc struktura nie jest artefaktem jednej metody |
| PCA | liniowość | tylko do wykresu (72% wariancji w 2 składowych); klastry liczone w pełnej przestrzeni 11 cech |

**Zasady, które stoją za wszystkimi wyborami:**
1. Najpierw dane, potem metoda (skośność → rangi, log, bootstrap; małe liczebności → testy dokładne).
2. Wielkość efektu i przedział ufności, a nie tylko p-value.
3. Najprostsza metoda, która nie jest gorsza (1-SE, LR zamiast boostingu, k = 4 zamiast 5).
4. Uczciwa walidacja (test użyty raz, nested CV, progi wybierane tylko na treningu).
5. Zrozumiałość dla biznesu.
6. Gdy założenie nie jest spełnione, mówię to wprost i dostosowuję metodę.

---

## Słowniczek: co oznaczają metryki

| Metryka | Po ludzku |
|---|---|
| **p-value** | prawdopodobieństwo uzyskania takiego lub bardziej skrajnego wyniku, *gdyby efektu nie było*. Nie mówi, jak duży jest efekt |
| **95% CI** | zakres wartości efektu zgodnych z danymi; szeroki CI oznacza, że mało wiemy |
| **Delta Cliffa** | od −1 do 1; jak często wartość z jednej grupy jest większa niż z drugiej. \|δ\| ≈ 0,15 to mały, 0,33 średni, 0,47 duży efekt |
| **RR / OR** | relative risk = stosunek odsetków (0,34 → trzy razy rzadziej); odds ratio = stosunek szans (stosowany w regresji logistycznej) |
| **h Cohena** | wielkość efektu dla dwóch proporcji (0,2 mały, 0,5 średni, 0,8 duży) |
| **Moc testu** | szansa, że test wykryje efekt, jeśli on istnieje. Standard to 80% |
| **MDE** | najmniejszy efekt, który test wykryje z zadaną mocą |
| **Holm** | korekta na wiele testów (kontroluje szansę choćby jednego fałszywego alarmu) |
| **CMH** | test różnicy proporcji w warstwach (np. kwartylach dochodu), czyli „po wyrównaniu” |
| **ROC-AUC** | szansa, że losowy responder dostanie wyższy wynik niż losowy nie-responder (0,5 = losowo) |
| **PR-AUC / AP** | jakość rankingu w tej części, którą faktycznie kontaktujemy; baza = odsetek pozytywów (0,15) |
| **Brier** | średni błąd kwadratowy prawdopodobieństw (im mniej, tym lepiej); mierzy kalibrację |
| **Lift @10%** | ile razy więcej responderów jest w top 10% niż średnio |
| **Nested CV** | strojenie w pętli wewnętrznej i ocena w zewnętrznej; uczciwa ocena *całej procedury* |
| **Reguła 1-SE** | wybierz najprostszy model, którego wynik mieści się w 1 błędzie standardowym od najlepszego |
| **Permutation importance** | o ile spada wynik, gdy „przetasujemy” kolumnę (grupę kolumn) |
| **Silhouette** | od −1 do 1; jak dobrze punkty pasują do swojego klastra w porównaniu z sąsiednim |
| **ARI** | zgodność dwóch podziałów na klastry (1 = identyczne); u mnie miara stabilności |
| **Próg rentowności** | 3/11 = 27,3%; kontakt opłaca się, gdy p(odpowiedź) > 27,3% |

---

## Pytania, które mogą paść na rozmowie, i krótkie odpowiedzi

- **„Dlaczego nie usunąłeś outlierów dochodu?”** – Tukey nie wskazał żadnych. Usunąłem tylko rekordy *logicznie niemożliwe* (6). Placeholdery 7 500 oznaczyłem i zostawiłem, bo nie zmieniają wniosków, a usuwanie „bo dziwne” to ryzyko selekcji.
- **„Dlaczego Mann-Whitney, a nie t-test?”** – skośność i dużo zer. Welcha i tak policzyłem jako sprawdzenie odporności, bo biznes patrzy na średnią (przychód).
- **„Dlaczego regresja logistyczna, a nie XGBoost?”** – nested CV pokazało, że LR jest lepsza (0,71 vs 0,64), a przy tym jest skalibrowana i łatwa do wyjaśnienia. Przy 291 pozytywach prostszy model wygrywa. XGBoost nie był na liście dozwolonych bibliotek; jego odpowiednikiem jest HistGradientBoosting.
- **„Dlaczego nie SMOTE?”** – PR-AUC mierzy ranking, więc oversampling go nie poprawia, za to psuje kalibrację prawdopodobieństw, a ta jest mi potrzebna do progu 27%. Wagi klas były w siatce; dla regresji logistycznej przegrały (las losowy je wybrał i ma najgorszą kalibrację).
- **„Skąd wiesz, że Z_Revenue to przychód?”** – to stałe, które dokładnie odtwarzają liczby z publicznego briefu iFood (6 720 i 3 674 MU). To mocna przesłanka, ale założenie warto potwierdzić u właściciela danych.
- **„Co byś zrobił, mając więcej czasu?”** – model uplift na danych z losowym holdoutem, ekonomia oparta na marży, GEE na 6 kampaniach (szczególnie dla Complain), potwierdzenie daty snapshotu (recency, wiek).
