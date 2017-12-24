# BOLT #1: Base Protocol

## Overview

Ten protokół obejmuje fundamentalne mechanizmy autentykacji i transportu, które odpowiadają za
formowanie indywidualnych wiadomości.
[BOLT #8](08-transport.md) opisuje w prostej formie warstwę transportu użytą w Lightning, która
może zostać zastąpiona przez każdą inną metodę transportu, która spełnia opisane poniżej wymagania.

Domyślny port TCP to 9735. W zapisie haxadecymalnym `0x2607`: kod znaku błyskawicy w Unicode.<sup>[1](#reference-1)</sup>

Wszystkie pola z danymi są big-endian chyba, że zaznaczono inaczej.

## Table of Contents
[TODO: Przetlumaczyc naglowki]
  * [Connection Handling and Multiplexing](#connection-handling-and-multiplexing)
  * [Lightning Message Format](#lightning-message-format)
  * [Setup Messages](#setup-messages)
    * [The `init` Message](#the-init-message)
    * [The `error` Message](#the-error-message)
  * [Control Messages](#control-messages)
    * [The `ping` and `pong` Messages](#the-ping-and-pong-messages)
  * [Acknowledgments](#acknowledgments)
  * [References](#references)
  * [Authors](#authors)

## Connection Handling and Multiplexing

Implementacje MUSZĄ używać pojedynczego połączenia dla każdego peera; channel messages (which include a channel ID) are multiplexed over this single connection. [TODO: Nie rozumiem]

## Format Wiadomości Lightning

Po odszyfrowaniu, wszystkie widomości Lightning mają taką postać:

1. `type`: 2-bajtowe pole big-endian field oznaczający typ wiadomości
2. `payload`: ładunek zmiennej długości, który obejmuje pozostałą część wiadomości w formacie pasującym do typu (`type`)

Pole `type` determinuje to, w jaki sposób zinterpretować pole `payload`.
Format dla każdego indywidualnego typu jest opisany przez specyfikację w tym repozytorium.
Typy podążąją za zasadją "w porządku być dziwnym", więc nody MOGĄ wysyłać typy o dziwnej numeracji bez ustalania czy odbiorca jest w stanie je zrozumieć.

Node wysyłający:
  - NIE MOŻE wysyłać an evenly-typed messagem która nie jest wylistowana tutaj bez wcześniejszej negocjacji. [TODO: A co z zasadą "w porządku być dziwnym"?

Node odbierający:
  - w momencie odebrania _odd_ wiadomości, nieznanego typu:
  	- MUSI zignorować otrzymaną wiadomość
  - w momencie odebrania _even_ wiadomości, nieznanego typu:
  	- MUSI fail the channel
    

Wiadomości są podzielone logicznie na cztery grupy, ułożone od najbardziej znaczącego bita czyli:

  - Ustawienia i kontrola (typy `0`-`31`): wiadomości związane z ustanawianiem połączenia, kontrolną, wspieranymi funkcjonalnościami i raportowaniem błędów (opisane niżej)
  - Kanały (typy `32`-`127`): wiadomości używane do ustanawiania i rozwiązywania kanałów mikropłatności (opisane w [BOLT #2](02-peer-protocol.md))
  - Commitment (typy `128`-`255`): wiadomości związane z aktualizowaniem transakcji commitment, a więc dodawanie, unieważnianie i uregulowywanie HTLCs. Oprócz tego aktualizowanie opłat transakcyjnych (fee) i wymienianie się podpisami (opisane w [BOLT #2](02-peer-protocol.md))
  - Routing (types `256`-`511`): wiadomości obejmujące powiadomienia z nodów i kanałów, aoraz aktywnej eksploracji ścieżki (opisane w [BOLT #7](07-routing-gossip.md)

Wymaga się by wielkość wiadomości zgodnie z wymaganiami warstwy transportowej musi zmieścić się w wymaganych 2 bajtach nieujemnej liczby całkowitej (unasigned int) Oznacza to, że największa możliwa długość wiadomości jest ograniczona do 65535 bajtów.

Wymagania co do noda:
  - MUSI ignorować wszelkie dodatkowe dane wykraczające poza długość, której oczekuje dla danego typu.
  - w momencie odebrania znanej wiadomości, której zawartość ma niedostateczną długość:
  	- MUST fail the channels
  - that understands an option in this specification: [TODO: Nie jestem pewien czy rozumiem]
  	- MUST include all the fields annotated with that option.

### Uzasadnienie

Domyślnie `SHA2` oraz publiczne klucze Bitcoina są zakodowane jako
big endian, więc byłoby nietopowo używać innej kolejności bajtów dla reszty pól.

Długość jest limitowana do 65535 bajtów przez kryptograficzną obudówkę (wrapping) i wiadomości w protokole nigdy nie mogą być dłuższe.

Zasada _w porządku być dziwnym_ pozwala na przyszłe, opcjonalne rozszerzenia bez negocnajcji lub specjalnych zmien klientów. Zasada "zignoruj dodatkowe dane" również pozwala na przyszłe rozszerzenia.

Implemetacje moga preferować by dane wiadomości były wyrównane do 8-bajtowych
obszarów (the largest natural alignment requirement of any type here);
jednak, dodawanie 6-bajtowego wypełniacza za typem pola zostało uznane za marnotrastwo: wyrównanie może być osiągnięte poprzez odszyfrowanie wiadomości do bufora z 6-bajtowym dopełnieniem.

## Wiadomości ustawień (setup)

### Wiadomość `init`

Gdy authentykacja jest wykonana, piewsza wiadomość ujawnia funkcjonalności wspierane lub wymagane przez tego noda, nawet jesli doszło jedynie do ponownego połączenia (reconnection).

[BOLT #9](09-features.md) zawiera listę globalnych i lokalnych funkcjonalności. Każda wiadomość jest generalnie reprezentowana jako `globalfeatures` or `localfeatures` przez 2 bity. Najmniej znaczący bit z numerem 0 oznacza _even_, drugi najbardziej znaczący bit z numerem 1 to _odd_.

Oba pola `globalfeatures` i `localfeatures` MUSZĄ być uzupełnione do pełncyh bajtów zerami.

1. type: 16 (`init`)
2. data:
   * [`2`:`gflen`]
   * [`gflen`:`globalfeatures`]
   * [`2`:`lflen`]
   * [`lflen`:`localfeatures`]

Dwubajtowe pola `gflen` i `lflen` oznaczają liczbę bajtwó pól, które poprzedzają.

#### Wymagania

Node wysyłający:
  - MUSI wysłać `init` jako pierwszą wiadomość Lightning dla każdego połączenia.
  - MUSI ustawić bity funcjonalności jak zdefiniowano w [BOLT #9](09-features.md).
  - Dla każdej niezdefiniowanej funkcjonalność MUSI ustawić bit na 0.
  - POWINIEN użyć minimalnych długości minimum lengths required to represent the feature fields.

Node odbierający:
  - MUSI czekać na odbiór `init` przed wysłaniem jakiejkolwiek innej wiadomości.
  - MUSI odpowiedzieć na bity znanych funkcjonalności jak opisano w [BOLT #9](09-features.md).
  - w momencie odebrania, bita funkcjonalności dziwnej (_odd_) które są niezerowe:
  	- MUSI zignorowac te bity
  - w momencie odebrania niezerowego bita funkcjonalności _even_:
  	- MUSI sfailować kanał.

#### Uzasadnienie

Opisana semantyka pozwala zarówno na wprowadzanie przyszłych, niekompatybilnych zmian oraz przyszłych zmian kompatybilnych wstecz. Bity generalnie powinny być przypisywane w parach po to, by opcjonalne funkcjonalności mogły stać się w przyszłości obowiązkowe.

Nody czekają na potwierdzenie funkcjonalności innych, by uprościć diagnostyke błędów gdy funkcjonalności sa niekompatybilne.

Maski funkcjonalności są rozdzielone na funkcjonalności lokalne (dotyczą jedynie protokołu pomiędzy tymi dwoma nodami) oraz funkcjonalności globalne (mogą wpływać na
HTLCs i są w związk z tym również rozgłaszane do innych nodów).

### Wiadomość `error`

Dla uproszczenia diagnostyki, często bardzo użytecznym jest, by powiedzieć peerowi, że coś jest nie tak.

1. type: 17 (`error`)
2. data:
   * [`32`:`channel_id`]
   * [`2`:`len`]
   * [`len`:`data`]

2-bajtowe pole `len` oznacza liczbę bajtów następnego pola "data"

#### Wymagania

Kanał jest idenstyfikowany poprzez `channel_id`, chyba, że `channel_id` wynosi 0 (wszystkie bajty ustawione na 0). W takim przypadku błąd odnosi się do wszystkich kanałów.

Node założycielski (funding node):
  - dla wszystkich wiadomości o "error" wysłanych przed  (lub w trakcie) wiadomości `funding_created`:
  	- MUSI użyć tymczasowego ID kanału ('temporary_channel_id') zamiast 'channel_id'.

The współzałożycielski (fundee node):
  - dla wszystkich wiadomości "error" wysłanych przed  (ale już nie w trakcie)  `funding_signed`:
  	- MUSI użyć tymczasowego ID kanału ('temporary_channel_id') zamiast 'channel_id'

Node wysyłający:
  - po wysłaniu `error`:
  	- MUSI sfailować kanał wskazany we wiadomości.
  - POWINIEN wysłać `error` przy niepoprawnym użyciu protokołu lub błędów wewnętrznych które powodują, że kanał staje się niestabilny lub prowadzą do przyszłych niestabilności komunikacji.
  - MOŻE wysłać puste pole `data`.
  - kiedy przyczyną błędu jest nieudane sprawdzanie podpisu:
  	- POWINIEN dołączyć surową, hex-encded transakcję w odpowiedzi do wiadomości 'funding_created`, `funding_signed`, `closing_signed`, lub `commitment_signed'  
  - kiedy `channel_id` wynosi 0:
  	- MUSI sfailować wszystkie kanały
  	- MUSI zamknąć wszystkie połączenia
  - MUSI ustawić `len` jako długość (ilość danych) w `data`.

Node odbierający:
  - w omencie otrzymywania wiadomości `error`:
  	- MUSI sfailować kanał do którego odnosi się wiadomość o błędzie.
  - jeśli wiadomość nie odnosi się do żadnego istniejącego kanału:
  	- MUSI zignorować wiadomość.
  - MUST truncate `len` to the remainder of the packet (if it's larger). [TODO]
  - jeśli `data` nie składa się jedynie z drukowalnych znaków ASCII (wartości byte z przedziału od 32 do 126 włącznie):
  	- NIE POWINIEN drukować 'data' wprost

#### Uzasadnienie

Błędy krytyczne wymagają przerwania konwersacji. Jeśli połączenie jest po prostu przerwanem wówczas peer może ponowić połączenie.
if the connection is simply dropped, then the peer may retry the
connection. Jest to użyteczne, by opisać naruszenie zasad protokołu dla diagnostyki, ponieważ to wskazuje,że peer ma błąd.

It may be wise not to distinguish errors in production settings, lest
it leak information — hence, the optional `data` field. [TODO]

## Wiadomości Kontroli

### Wiadomości `ping` i `pong`

By możliwe było tworzenie połączeń TCP trwających bardzo długo, od czasu do czasu może być wymagane, by obie strony utrzymały żywe połączenie w warstiwe aplikacji. Takie wiadomości umożliwiają również zaciemnianie wzroców ruchu (traffic patterns).

1. type: 18 (`ping`)
2. data:
    * [`2`:`num_pong_bytes`]
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

Wiadomość `pong` jest wysyłana gdy odebrano wiadomość `ping`. Służy jako odpowiedź oraz utrzymanie połączenia przy życiu oraz wyraźne poinformowanie drugiej strony, że odbiorca ciągle żyje. W ramach odebranej wiadomości `ping`, wysyłający będzie określał liczbę bajtów, która ma być załączona jako zawartość wiadomości `pong`.

1. type: 19 (`pong`)
2. data:
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

#### Wymagania

Node wysyłający wiadomość `ping`:
  - POWINIEN wypełnić pole `ignored` zerami.
  - NIE MOŻE wypełnić `ignored` wrażliwymi danymi takimi jak np. klucze prywatne czy porcje danych inicjalizacyjnych z pamięci.
  - jeśli nie odbierze w odpowiedzi wiadomości `pong`:
  	- MOŻE zerwać połączenie sieciowe
  		- i MUSI sfailować kanał w tym przypadku
  - NIE POWINIEN wysyłać wiadomości `ping` częściej niż co 30 sekund.

Node wysyłający wiadomość `pong`:
  - POWINIEN wypełnić pole `ignored` zerai.
  - NIE MOŻE wypełnić `ignored` wrażliwymi danymi takimi jak np. klucze prywatne czy porcje danych inicjalizacyjnych z pamięci.

Node odbierający wiadomość `ping`:
  - POWINIEN sfailować kanał jeśli otrzymał znacznie więcej niż jeden `ping` na 30 sekund.
  - jeśli `num_pong_bytes` wynosi mniej niż 65532:
  	- MUSI odpowiedzieć wysyłając wiadomość `pong` z polem 'byteslen' równym polu 'num_pong_bytes'.
  - w przeciwnym wypadku (`num_pong_bytes` **nie** wynosi mniej niż 65532):
  	- MUSI zignorować `ping`.

Node odbierający wiadomość `pong`:
  - jeśli `byteslen` nie pasuje do pola  `num_pong_bytes` z jakiegokolwiek wysłanego `ping`'a:
  	- MOŻE sfailować kanał.

### Uzasadnienie

Największa możliwa wielkość wiadomości to 65535 bytes; więc maksymalna sensowna wartość pola `byteslen` to 65531 — Ponieważ należy odjąć wielkość pola `pong` i `byteslen`. This allows a convenient cutoff for `num_pong_bytes` to indicate that no reply should be sent. [TODO]

Połączenia między nodami w ramach sieci mogą trwać bardzo długo, jako że kanały płatności mają niezdefiniowaną długość życia. Jest jednak prawdopodobne, że żadne dane nie będą przesyłane przez istotną część czasu życia połączenia. Poza tym,na niektórych platformach jest możliwe, że klienci Lightning będą przełączane w tryb uśpienia bez uprzedniego streżenia. Dlatego odrębna wiadomość `ping` może być użyta zarówno w celu sprawdzenia, czy druga strona nadal żyje jak i w celu utrzymania połączenia przy życiu.

Dodatkowo możliwość wysłania wraz z wiadomością określonej liczby bajtów umożliwia nodom w sieci na tworzenie _syntetycznego_ ruchu. Taki sztuczny ruch może być częściowo użyty przeciwko próbom analizowania ruchu w oparciu o pakiety oraz czasy ich przesyłania. Nody mogą generować sztuczny ruch podobny do tego prawdziwego bez powodowania jakichkolwiek prawdziwych zmian w swoich kanałach.

W połączeniu z routwaniem cebulowym zdefiniowanym w
[BOLT #4](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md),
ostrożnie użyty sztuczny ruch może służyć jako wzmocnienie prywatności uczestników sieci.

Zaleca się ostrożność w stosowaniu komend `ping`, jako że pewna tolerancja powstaje w wyniku opóźnień działania sieci. Zwróć uwagę, że są inne metody na zalewanie ruchu przychodzącego (np. wysyłanie -odd- nieznanych typów wiadomości lub wypełnianie każdej wiadomości maksymalnie)
Limited precautions are recommended against `ping` flooding, however some
latitude is given because of network delays. Note that there are other methods
of incoming traffic flooding (e.g. sending _odd_ unknown message types, or padding
every message maximally). [TODO]

W końcu, okresowe użycie wiadomości `ping` służy do propagowania częstej rotacji kluczy jak opisano w [BOLT #8](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md).

## Podziękowania

[ TODO: (roasbeef); fin ]

## Odnośniki

1. <a id="reference-2">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Autorzy

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
