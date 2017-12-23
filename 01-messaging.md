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
  - dla wszystkich wiadomości o "error" wysłanych przed  (ale już nie w trakcie)  `funding_signed`:
  	- MUSI użyć tymczasowego ID kanału ('temporary_channel_id') zamiast 'channel_id'

Node wysyłający:
  - po wysłaniu `error`:
  	- MUSI sfailować kanał wskazany we wiadomości.
  - POWINIEN wysłać `error` przy niepoprawnym użyciu protokołu lub błędów wewnętrznych które powodują, że kanał staje się niestabilny lub prowadza do przyszłych niestabilności komunikacji.
  - MOŻE wysłać puste pole `data`.
  - when failure was caused by an invalid signature check:
    - SHOULD include the raw, hex-encoded transaction in reply to a `funding_created`, `funding_signed`, `closing_signed`, or `commitment_signed` message.
  - when `channel_id` is 0:
    - MUST fail all channels.
    - MUST close the connection.
  - MUST set `len` equal to the length of `data`.

The receiving node:
  - upon receiving `error`:
    - MUST fail the channel referred to by the error message.
  - if no existing channel is referred to by the message:
    - MUST ignore the message.
  - MUST truncate `len` to the remainder of the packet (if it's larger).
  - if `data` is not composed solely of printable ASCII characters (For reference: the printable character set includes byte values 32 through 126, inclusive):
    - SHOULD NOT print out `data` verbatim.

#### Rationale

There are unrecoverable errors that require an abort of conversations;
if the connection is simply dropped, then the peer may retry the
connection. It's also useful to describe protocol violations for
diagnosis, as this indicates that one peer has a bug.

It may be wise not to distinguish errors in production settings, lest
it leak information — hence, the optional `data` field.

## Control Messages

### The `ping` and `pong` Messages

In order to allow for the existence of very long-lived TCP connections, at
times it may be required that both ends keep alive the TCP connection at the
application level. Such messages also allow obfuscation of traffic patterns.

1. type: 18 (`ping`)
2. data:
    * [`2`:`num_pong_bytes`]
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

The `pong` message is to be sent whenever a `ping` message is received. It
serves as a reply and also serves to keep the connection alive, while
explicitly notifying the other end that the receiver is still active. Within
the received `ping` message, the sender will specify the number of bytes to be
included within the data payload of the `pong` message.

1. type: 19 (`pong`)
2. data:
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

#### Requirements

A node sending a `ping` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
memory.
  - if it doesn't receive a corresponding `pong`:
    - MAY terminate the network connection,
      - and MUST NOT fail the channels in this case.
  - SHOULD NOT send `ping` messages more often than once every 30 seconds.

A node sending a `pong` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
 memory.

A node receiving a `ping` message:
  - SHOULD fail the channels if it has received significantly in excess of one `ping` per 30 seconds.
  - if `num_pong_bytes` is less than 65532:
    - MUST respond by sending a `pong` message, with `byteslen` equal to `num_pong_bytes`.
  - otherwise (`num_pong_bytes` is **not** less than 65532):
    - MUST ignore the `ping`.

A node receiving a `pong` message:
  - if `byteslen` does not correspond to any `ping`'s `num_pong_bytes` value it has sent:
    - MAY fail the channels.

### Rationale

The largest possible message is 65535 bytes; thus, the maximum sensible `byteslen`
is 65531 — in order to account for the type field (`pong`) and the `byteslen` itself. This allows
a convenient cutoff for `num_pong_bytes` to indicate that no reply should be sent.

Connections between nodes within the network may be very long lived, as payment
channels have an indefinite lifetime. However, it's likely that
no new data will be
exchanged for a
significant portion of a connection's lifetime. Also, on several platforms it's possible that Lightning
clients will be put to sleep without prior warning. Hence, a
distinct `ping` message is used, in order to probe for the liveness of the connection on
the other side, as well as to keep the established connection active.

Additionally, the ability for a sender to request that the receiver send a
response with a particular number of bytes enables nodes on the network to
create _synthetic_ traffic. Such traffic can be used to partially defend
against packet and timing analysis — as nodes can fake the traffic patterns of
typical exchanges without applying any true updates to their respective
channels.

When combined with the onion routing protocol defined in
[BOLT #4](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md),
careful statistically driven synthetic traffic can serve to further bolster the
privacy of participants within the network.

Limited precautions are recommended against `ping` flooding, however some
latitude is given because of network delays. Note that there are other methods
of incoming traffic flooding (e.g. sending _odd_ unknown message types, or padding
every message maximally).

Finally, the usage of periodic `ping` messages serves to promote frequent key
rotations as specified within [BOLT #8](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md).

## Acknowledgments

[ TODO: (roasbeef); fin ]

## References

1. <a id="reference-2">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
