# BOLT #0: Wstęp i spis treści

Witaj przyjacielu! Ten dokument
opisuje protokół drugiej warstwy dla bitcoinowych transakcji off-chain
poprzez wzajemną kooperację, polegając na transakcjach on-chain jeśli to konieczne
jako metoda egzekwowania płatności. 

Niektóre wymagania są subtelne; staraliśmy się podkreślić motywacje
i powody które stoją za rezultatami, które widzisz. Jeśli widzisz jakąś
część dezorientującą lub błędną, proszę skontaktuj się z nami i pomóż ulepszyć dokument.

To jest wersja 0.

1. [BOLT #1](01-messaging.md): Podstawowe dane o protokole
2. [BOLT #2](02-peer-protocol.md): Protokół P2P oraz zarządzanie kanałami.
3. [BOLT #3](03-transactions.md): Format bitcoinowych trasakcji oraz skryptów
4. [BOLT #4](04-onion-routing.md): Protokół routowania cebulowego
5. [BOLT #5](05-onchain.md): Rekomendacje dotyczace wyłapywania transakcji on-chain
7. [BOLT #7](07-routing-gossip.md): Wyszukiwanie nodów i kanałów P2P
8. [BOLT #8](08-transport.md): Encrypted and Authenticated Transport [TODO]
9. [BOLT #9](09-features.md): Przypisywanie flag dot. funkcjonalności
10. [BOLT #10](10-dns-bootstrap.md): DNS Bootstrap and Assisted Node Location [TODO]
11. [BOLT #11](11-payment-encoding.md): Protokół Invoice dla płatności Lightning

## Słowniczek i terminologia

* *Node*:
   * Komputer lub inne urządzenie połączone do noda bitcoin.

* *Peers*:
   * *Nodey* transferujący bitcoiny między soba przez *kanał*.

* *MSAT*:
   * Millisatoshi, zazwyczaj używane jako nazwa pola.

* *Funding transaction*:
   * Niewycofywalna transakcja on-chain która wysyla środki do obu *peerów* on a *kanale*.
   Mogą być wydane jedynie za obupólną zgodą.

* *Kanal*:
   * Szybka metoda płatności off-chain poprzez obupólną wymianę pomiędzy *peerami*.
   By dokoncać transakcji, peery wymieniają się podpisami, by stworzyć uaktualnienie do *commitment transaction*.

* *Transakcja commitment*:
   * Transakcja, która wydaje *funding transaction*.
   Każdy *peer* przechowuje podpisdrugiego peera dla tej transakcji, więc
   każdy z nich zawsze posiada commitment transaction, która może być wydana.
   Gdy nowa transakcja commitment jest ustalona, stara zostaje *unieważniona*.

* *HTLC*: Hashed Time Locked Contract.
   * Warunkowa płatność pomiędzy dwoma *peerami*: odbiorca może użyć płatności
    poprzez zaprezentowanie jej podpisu oraz *preimage*,
    w przeciwnym razie płatnik może anulować płatność poprzez wydanie jej 
	po upłynięciu określonego czasu. Są one zaimplementowane jako wyjścia dla
    *transakcji commitment*.

* *hash płatności*:
   * Hash płatności zawarty w kontrakcie *HTLC*. Jest to hash z *payment preimage*.

* *Payment preimage*:
   * Dowód, że płatność została odebrana, przetrzymywany przez finalnego odbiorcę,
    który jest jedyna osobą, która zna ten sekret. Zwany też sekretną liczbą R.
    Finałowy odbiorca ujawnia preimage w celu uwolnienia funduszy.
    Preimage jest zahsachowany jako *payment hash* in the *HTLC*.

* *Commitment revocation secret key*:
   * Every *commitment transaction* has a unique *commitment revocation* secret-key
    value that allows the other *peer* to spend all outputs
    immediately: revealing this key is how old commitment
    transactions are revoked. To support revocation, each output of the
    commitment transaction refers to the commitment revocation public key.

* *Per-commitment secret*:
   * Każda *transakcja commitment* bierze swoje klucze z per-commitment secret, [TODO]
     która jest generowana tak, że seria per-commitment secrets dla wszystkich wcześniejszych
     commitmentów może być zapisana w sposób kompaktowy [TODO: shachain?]

* *Obupólne zamknięcie*:
   * Zamknięcie *kanału* przy kooperacji obu stron, zakończone rozesłaniem an unconditional
    spend of the *funding transaction* with an output to each *peer*
    (unless one output is too small, and thus is not included). [TODO]

* *Jednostronne zamknięcie*:
   * Zamknięcie *kanału bez porozumienia z druga strona*, zakończone wysłaniem
    *transakcji commitment*. Ta transakcja jest większa (np. mniej efektywna)
    niż transakcja *obupólnego zamknięcia* i strona wysylające tą transakcję
    nie może podjąć swoich środków przez ustalony wcześniej czas.

* *Zamknięcie transakcją unieważnioną*:
   * Nieprawidłowe zamknięcie *kanału*, zakończone wysłaniem unieważnionej
    *transakcji commitment*. Ponieważ drugi *peer* zna
    *commitment revocation secret key*, może on zostać użyty do *transakcji karnej*.

* *Transakcja karna*:
   * Transakcja, która wydaje wszystkie outputy z transakcji unieważnionej, która została
   zacomitowana, dzięki użyciu *commitment revocation secret key*. *peer* używa jej jeśli
   drugi peer próbuje go "oszukać" poprzez rozesłanie transakcji unieważnionej.

* *Numer Commitmentu*:
   * 48-bitowy licznik, który zwiększa się dla każdej *transakcji commitment*; liczniki są niezależne dla każdego *peera* w *kanale* i startują od 0.

* *Można być nietypowym*: [TODO: Nie rozumiem]
   * Zasada zastosowana do niektórcyh pól numerycznych która wskazuje na istnienie
     opcjonalnego lub obowiązkowego wsparcia dla funkcjonalności.
     Even numbers indicate that both endpoints
     MUST support the feature in question, while odd numbers indicate
     that the feature MAY be disregarded by the other endpoint.

* `chain_hash`:
   * Użyty w kilku dokumentach BOLT by oznaczyć genesis hash of a
     target blockchain. To pozwala *nodom* tworzyć i powiązywać *kanały* 
	na różnych blockchainach. Nody powinny ignorować wszystkie wiadomości, które
	odwołują się do `chain_hash` który nie jest im znany. W przeciwieństwie do 
	`bitcoin-cli`, hash nie jest odwrócony, tylko użyty bezpośrednio. [TODO:?]

     Dla głównego łańcucha Bitcoin, `chain_hash` MUSI wynosić
     (hexadecymalnie):
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.

## Theme Song

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
