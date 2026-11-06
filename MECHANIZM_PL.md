# Knowledge Assistant: jak to działa (notatki do nauki)

Projekt na zmyślonych danych. Produkt „NotifyHub” nie istnieje, artykuły pomocy i pytania testowe są wymyślone.

## W jednym zdaniu

Asystent odpowiada na pytania klienta wyłącznie na podstawie artykułów pomocy, przy każdej odpowiedzi pokazuje, z którego fragmentu ją wziął, a jeśli artykuły tego nie mówią, odpowiada „I don't know based on the documentation” i zapisuje pytanie jako lukę w dokumentacji.

## Czym jest RAG

**RAG** (Retrieval-Augmented Generation, czyli „generowanie wsparte wyszukiwaniem”) to sposób używania AI w dwóch krokach:

1. **Wyszukaj** (retrieval): znajdź w dokumentach kilka fragmentów, które pasują do pytania.
2. **Wygeneruj** (generation): daj AI pytanie razem z tymi fragmentami i każ odpowiedzieć tylko na ich podstawie.

Porównanie: zwykły czat z AI to egzamin z pamięci. Model odpowiada z tego, czego się kiedyś nauczył, i bywa, że zmyśla. RAG to egzamin z otwartą książką, ale z jedną zasadą: wolno pisać tylko to, co jest na otwartych stronach, i trzeba podać numer strony.

Po co to firmie? AI nie zna twojego produktu, cennika ani ustawień. Dokumentacja je zna. RAG łączy jedno z drugim, a podane źródła pozwalają każdemu sprawdzić odpowiedź w 10 sekund.

## Droga jednego pytania, węzeł po węźle

| Krok | Węzeł w n8n | Co robi | Po co |
| --- | --- | --- | --- |
| 1 | `When chat message received` | Przyjmuje pytanie z czatu n8n | Wejście dla człowieka |
| 1b | `When called by eval` | Przyjmuje pytanie z workflow testowego | Test używa dokładnie tej samej ścieżki co czat |
| 2 | `Normalize question` | Z obu wejść robi to samo pole `question` i zapisuje, skąd przyszło (`chat` albo `eval`) | Dalsze węzły nie muszą wiedzieć, skąd przyszło pytanie |
| 3 | `Get KA Docs` | Czyta 43 fragmenty artykułów z tabeli `KA Docs` | To jest „baza wiedzy” |
| 4 | `Search docs` (Code) | Ocenia każdy fragment od 0 do 1 i zostawia 4 najlepsze | Do AI trafia tylko to, co pasuje do pytania |
| 5 | `Good enough match?` (IF) | Sprawdza, czy najlepszy fragment ma ocenę co najmniej 0,35 | Bez pasującego fragmentu nie ma sensu pytać AI |
| 5a | `No match: I don't know` | Gdy ocena jest za niska: od razu „nie wiem”, status `no_match` | Zero ryzyka zmyślenia, zero kosztu AI |
| 6 | `Ask Claude` | Claude Haiku dostaje pytanie i 4 fragmenty, oddaje JSON: `answer`, `sources`, `not_covered` | Ułożenie odpowiedzi ludzkim językiem |
| 7 | `Check citations` (Code) | Sprawdza odpowiedź AI kodem, nie wiarą | Ostatnia bramka przed klientem |
| 8 | `Save to KA Answers` | Zapisuje każde pytanie: pytanie, odpowiedź, źródła, znalezione fragmenty, ocenę, status | Historia i lista luk w dokumentacji |
| 9 | `Reply` | Składa odpowiedź do czatu: tekst + „Sources: D04-1” | To widzi użytkownik |

### Jak liczona jest ocena w kroku 4

1. Z pytania wyrzucam słowa bez treści („the”, „how”, „is”…) i skracam końcówki („segments” → „segment”).
2. Każde słowo dostaje wagę: im rzadziej występuje w dokumentach, tym jest ważniejsze. „Segment” pojawia się w kilku fragmentach, więc dużo mówi. Słowo, które jest wszędzie, mówi mało. To się nazywa **IDF**.
3. Ocena fragmentu = suma wag słów z pytania, które w nim są, podzielona przez sumę wag wszystkich słów z pytania.

Przykład: pytanie „How often are dynamic segments recalculated?” ma ważne słowa: often, dynamic, segment, recalculat. Fragment D04-1 zawiera trzy z nich, w tym rzadkie „recalculat”, więc dostaje 0,655 i wygrywa.

### Co sprawdza `Check citations` w kroku 7

1. Czy odpowiedź AI to poprawny JSON? Jeśli nie, to „nie wiem”, status `ai_error`.
2. Czy AI samo powiedziało `not_covered: true`? Wtedy „nie wiem”, status `not_covered`.
3. Czy jest jakaś odpowiedź i jakieś źródło? Jeśli nie, to „nie wiem”, status `citation_failed`.
4. Czy **każde** podane źródło jest wśród 4 znalezionych fragmentów? Jedno obce i cała odpowiedź odpada: „nie wiem”, status `citation_failed`.
5. Wszystko w porządku: status `answered`.

Zasada nazywa się **fail closed**: przy każdej wątpliwości system wybiera bezpieczną odpowiedź „nie wiem”, a nie „jakąś”.

## Słowa kluczowe czy wektory: różnica

| | Wyszukiwanie po słowach kluczowych (tu) | Wyszukiwanie wektorowe |
| --- | --- | --- |
| Jak działa | Liczy wspólne słowa pytania i fragmentu | Zamienia teksty na liczby opisujące znaczenie (embeddingi) i szuka podobnych |
| Synonimy („cost” i „price”) | Nie złapie | Złapie |
| Koszt | Darmowe, zwykły kod | Płatne wywołania modelu do embeddingów i baza wektorowa |
| Powtarzalność | Ten sam wynik zawsze | Prawie zawsze ten sam |
| Łatwość wyjaśnienia | Widać, które słowa pasowały | „Te teksty są blisko w przestrzeni wektorowej” |

Dlaczego tu słowa kluczowe: w n8n nie mam własnych danych logowania ani podłączonej bazy wektorowej. Claude działa na wbudowanych kredytach n8n. Przy 14 krótkich artykułach słowa kluczowe wystarczyły, a wynik da się wytłumaczyć linijka po linijce. Przy dużej, chaotycznej dokumentacji następnym krokiem byłyby wektory.

## Dlaczego próg

Wyszukiwarka zawsze coś zwróci, nawet gdy nic nie pasuje. Gdyby AI dostało słabe fragmenty, kusiłoby je, żeby „coś” odpowiedzieć. Próg 0,35 odcina takie przypadki przed AI. W teście 3 z 5 pytań bez odpowiedzi (SMS, SLA, HIPAA) odpadły właśnie na progu.

Uczciwie: próg wybrałem, patrząc na oceny tych samych 15 pytań. Pytania z odpowiedzią miały co najmniej 0,496, pytania bez odpowiedzi poniżej progu miały najwyżej 0,217. Przy prawdziwej dokumentacji trzeba by go sprawdzić na większym zestawie.

## Dlaczego kontrola cytowań

Model językowy potrafi napisać „Source: D09-2”, chociaż nigdy nie widział D09-2. Prośba w prompcie „cytuj tylko podane fragmenty” to tylko prośba. Kod w `Check citations` to sprawdza i w razie wątpliwości blokuje odpowiedź. Sprawdziłem to testem: podsunąłem workflowowi sztuczną odpowiedź AI ze źródłem D09-2, którego wyszukiwarka nie znalazła, i odpowiedź została zastąpiona przez „nie wiem”.

Dwie warstwy razem: próg łapie pytania zupełnie obok tematu, a AI z flagą `not_covered` łapie podchwytliwe pytania, które mają wspólne słowa z dokumentami, ale nie mają w nich odpowiedzi (w teście: limit automatyzacji i limit powiadomień dziennie).

## Co mówi test

| Miara | Wynik (oba przebiegi) | Co znaczy |
| --- | --- | --- |
| Trafność wyszukiwania | 10 z 10 | Właściwy artykuł był wśród 4 znalezionych fragmentów |
| Poprawne cytowania | 10 z 10 | AI odpowiedziało i powołało się na fragment z właściwego artykułu |
| Kluczowy fakt | 10 z 10 | W odpowiedzi była właściwa liczba albo nazwa, np. „15 minutes” |
| Poprawne odmowy | 5 z 5 | Na pytania spoza dokumentacji padło „nie wiem” (3 przez próg, 2 przez AI) |
| Fałszywe odmowy | 0 | Ani razu „nie wiem” na pytanie, na które odpowiedź jest |

Czego test **nie** dowodzi: że system zadziała tak samo na prawdziwej, dużej dokumentacji. Artykuły i pytania napisałem sam, jest ich mało, a próg dobrałem do tych pytań.

## Słowniczek

- **RAG**: wyszukaj fragmenty, potem każ AI odpowiedzieć tylko z nich.
- **Chunk (fragment)**: kawałek dokumentu, tu jedna sekcja artykułu. AI dostaje fragmenty, nie całe artykuły.
- **chunk_id**: identyfikator fragmentu, np. D04-1 = artykuł D04, sekcja 1.
- **Retrieval (wyszukiwanie)**: krok, w którym system wybiera fragmenty pasujące do pytania.
- **Top-k**: ile najlepszych fragmentów bierzemy. Tu k = 4.
- **IDF**: waga słowa. Rzadkie słowo jest ważniejsze niż słowo, które jest wszędzie.
- **Próg (threshold)**: minimalna ocena, poniżej której nie pytamy AI.
- **Embedding**: zamiana tekstu na listę liczb opisującą jego znaczenie. Podstawa wyszukiwania wektorowego.
- **Halucynacja**: AI pewnym tonem podaje coś, co nie jest prawdą albo nie wynika ze źródeł.
- **Grounding (ugruntowanie)**: odpowiedź oparta na konkretnym źródle, które da się sprawdzić.
- **Fail closed**: przy błędzie system wybiera bezpieczne wyjście („nie wiem”), a nie zgadywanie.
- **Prompt injection**: próba przemycenia poleceń dla AI w treści pytania albo dokumentu. Dlatego prompt mówi, że pytanie i fragmenty to dane, nie polecenia.
- **Eval (ewaluacja)**: automatyczny test z oczekiwanymi wynikami, który można powtarzać po każdej zmianie.
- **Fałszywa odmowa**: „nie wiem” na pytanie, na które odpowiedź jest w dokumentach.
- **Luka w dokumentacji**: pytanie, na które dokumenty nie odpowiadają. Tu: każdy wiersz w `KA Answers` ze statusem innym niż `answered`.

## Jak to opowiedzieć w 60 sekund (po angielsku)

> In onboarding and support, customers ask the same product questions every day, and the answers are already in the help center. I built a documentation assistant in n8n that answers only from those articles and always shows the source.
>
> It works in two steps. First, a search step finds the four most relevant sections of the docs. If nothing matches well enough, it says "I don't know" without calling the AI at all. Second, Claude writes the answer from those sections only and returns the section IDs it used. Then code checks that every cited section was actually retrieved. If not, the answer is blocked.
>
> Every question is logged, so unanswered ones become a list of documentation gaps.
>
> I tested it on fictional data with 15 questions: 10 with answers in the docs and 5 without. It found the right article 10 out of 10 times, cited it correctly 10 out of 10, and refused all 5 it should have refused, with no false refusals. It's a small test on data I wrote myself, but it shows I design AI tools that are safe to put in front of customers.

## 5 pytań rekrutera z odpowiedziami

**1. Why did you add an "I don't know" answer? Isn't it better to always answer?**
In support, a confident wrong answer costs more than no answer. The customer acts on it, and the team has to fix it later. "I don't know" plus a logged gap is honest, and it tells the team which article to write next.
*(Po polsku: zła odpowiedź podana pewnym tonem kosztuje więcej niż „nie wiem”.)*

**2. How do you know the AI isn't making up sources?**
I don't trust the prompt alone. After the AI answers, code checks that every cited section ID is one of the sections the search actually returned. One unknown ID and the answer is replaced with "I don't know". I tested it with a fake reply that cited a section that wasn't retrieved, and it was blocked.
*(Kod sprawdza każde źródło. Sprawdziłem to sztuczną odpowiedzią z fałszywym źródłem.)*

**3. Why keyword search and not vector search?**
For 14 short articles it was enough, it costs nothing, and I can explain every result. The trade-off is synonyms: "price" and "cost" don't match. With a large help center I'd add embeddings and compare both on the same test set.
*(Wystarczyło na tę skalę, jest darmowe i przejrzyste. Słabość: synonimy.)*

**4. How did you test it, and what are the limits of your test?**
A separate evaluation workflow sends 15 questions through the real workflow: 10 answerable with an expected article and fact, 5 not covered, two of them deliberately tricky. It measures retrieval, citations, key facts and refusals. Both runs scored 10/10, 10/10, 10/10 and 5/5. The limits: I wrote both the docs and the questions, the set is small, and the threshold was chosen on the same questions.
*(Najpierw wynik, potem uczciwie ograniczenia. Rekruterzy to cenią.)*

**5. How would you take this to production?**
Connect the real help center and re-split it into sections, add vector search, build a test set of at least 50 real customer questions, re-tune the threshold, and have a person review answers at first. I'd also send the documentation owner a weekly list of unanswered questions.
*(Prawdziwe dokumenty, wektory, większy test, człowiek sprawdza na początku, raport luk co tydzień.)*

## Mini-test (10 pytań)

Zakryj odpowiedzi i spróbuj sam.

1. Co oznacza skrót RAG i z jakich dwóch kroków się składa?
2. Czym jest chunk i jaki ma identyfikator w tym projekcie?
3. Co się dzieje, gdy najlepszy fragment ma ocenę 0,2?
4. Dlaczego w kroku wyszukiwania rzadkie słowa ważą więcej?
5. Jakie trzy pola zwraca Claude w JSON?
6. Co robi `Check citations`, gdy AI powoła się na fragment, którego wyszukiwarka nie znalazła?
7. Co to znaczy „fail closed”?
8. Które dwa pytania testowe przeszły przez próg, a mimo to dostały „nie wiem”? Kto im odmówił?
9. Jaka jest główna słabość wyszukiwania po słowach kluczowych w porównaniu z wektorowym?
10. Gdzie szukać luk w dokumentacji?

<details>
<summary>Odpowiedzi</summary>

1. Retrieval-Augmented Generation. Najpierw wyszukaj pasujące fragmenty, potem każ AI odpowiedzieć tylko z nich.
2. Kawałek dokumentu, tu jedna sekcja artykułu. Identyfikator to `chunk_id`, np. D04-1.
3. Jest poniżej progu 0,35, więc workflow od razu odpowiada „nie wiem” ze statusem `no_match`, bez pytania AI.
4. Bo słowo, które występuje prawie wszędzie, nie pomaga odróżnić fragmentów. Rzadkie słowo mocno wskazuje właściwy fragment (IDF).
5. `answer`, `sources`, `not_covered`.
6. Zastępuje całą odpowiedź przez „I don't know based on the documentation.” i zapisuje status `citation_failed`.
7. Przy błędzie albo wątpliwości system wybiera bezpieczną odpowiedź, tutaj „nie wiem”, zamiast zgadywać.
8. N02 (limit automatyzacji) i N04 (limit powiadomień dziennie). Odmówił sam Claude, ustawiając `not_covered: true`.
9. Nie łapie synonimów i parafraz bez wspólnych słów, np. „cost” i „price”.
10. W tabeli `KA Answers`: każdy wiersz ze statusem innym niż `answered` (`no_match`, `not_covered`, `citation_failed`, `ai_error`).

</details>
