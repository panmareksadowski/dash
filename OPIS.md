# Zmiany
Główne zmiany będą się odbywać w pliku [chainparams.cpp](https://github.com/panmareksadowski/dash/blob/master/src/chainparams.cpp), gdzie zakodowana jest więkoszość parametrów sieci i parametrów blockchain. Oprócz parametrów specyficznych dla Dash na przykład dotyczących <code>instantsend</code>.
### Utworzenie nowej sieci P2P
<https://github.com/panmareksadowski/dash/commit/cdee1be7e57018c1e2ed6badd1622b5f81f17901>

Zmiana portu(<code>nDefaultPort</code>) na którym nasłuchują elementy sieci P2P(zarówno zwykłe node'y jak i masternode'y) na <code>18999</code>(może być wybrany dowolnie)

Wyczyszczenie <code>vSeeds</code> zawierającego serwery DNS z adresami node'ow i masternod'ow.
Wyczyszczenie <code>vFixedSeeds</code> i dodanie adresu serwera na którym będzie uruchomiony przeze mnie node.
Włączający się do sieci node będzie najpierw próbował się połączyć z node'ami z którymi się połączył w poprzedniej sesji zapisanymi w pliku <code>peers.dat</code> w folderze <code>.dashcore</code>. Jeśli takich nie ma(bo na przykład łączymy się z siecią po raz pierwszy) lub nie uda się połączenie z nimi będzie próbował zapytać się o adresy z którymi mógłby się połączyć serwerów DNS zakodowanych w <code>vSeeds</code>. Jeśli to się nie powiedzie(dla naszej sieci zawsze się nie powiedzie, bo <code>vSeeds</code> jest puste) będzie próbował się połączyć z adresami zakodowanymi w <code>vFixedSeeds</code>(tutaj wpisaliśmy adres naszego serwera).
Adresy serwerów dla oryginalnej sieci dash są zapisane w pliku [chainparamsseeds.h](https://github.com/panmareksadowski/dash/blob/master/src/chainparamsseeds.h).
Po podłączeniu do sieci wierzchołki wymieniają się informacjami między sobą o swoich adresach.

<code>pchMessage</code> zawiera pierwsze ustalone 4 bajty nagłówka wiadomości wysyłanych między node'ami. Zmieniamy je na inną wartość dla naszej sieci. Dzięki temu można rozpoznać czy wiadomość pochodzi na pewno z naszej sieci.(Wiadomości w mainet dash będą wyglądały tak samo, ale będą się różniły tymi pierwszymi 4 bajtami.)

Ustawienie parametrów sprawdzających, czy pobraliśmy właściwy początek blockchain. Ustawiamy na genesis blok, bo to jedyny blok jaki znamy.

Po tych zmianach możemy już utworzyć działającą niezależną nową sieć(nową kryptowalutę).

### Zmiana "Genesis block"

<https://github.com/panmareksadowski/dash/commit/a6ed18e0d34e4b9da8e00cf255c9484ae120f4a4>

Zmiana genesis block.

Zmiana oczekiwanego czasu wykopania bloku(<code>consensus.nPowTargetSpacing</code>) na 1 minutę oraz czasu zmiany trudności(<code>consensus.nMinerConfirmationWindow</code>) na co 120 bloków, czyli około 2 godzin (<code>consensus.nPowTargetTimespan</code>).

Zmieniamy parametrów trudności(<code>genesis.nBits</code> zakodowane w bloku i consensus.powLimit) tak, żeby można było łatwo i szybko wykopać pierwsze bloki.

Ponieważ zmieniliśmy genesis blok musimy go wykopać(znaleźć jego hash, który będzie spełniał warunek ustalonej trudności):

     /**
     * Mine genesis block
     */
    arith_uint256 hashTarget = arith_uint256().SetCompact(genesis.nBits);

    //find nNonce and nTime which fulfill hashTarget
    uint256 hash = genesis.GetHash();
    while (UintToArith256(hash) > hashTarget)
    {
        genesis.nNonce++;
        if(genesis.nNonce == 0) //if nNonce wrapped, try change time
            genesis.nTime++;
        hash = genesis.GetHash();
    }

    consensus.hashGenesisBlock = genesis.GetHash();

Usuwamy asercje porównujące hash bloku i hash merkle tree(zmieniliśmy transakcje coinbase w bloku, więc merkle tree też się zmieniło) z zakodowanymi wartościami.

    //don't need this checks we have calculated propely hashes
    //if we want we can first calculate properly hashes and then hardcode them here
    //assert(consensus.hashGenesisBlock == uint256S("0x00000ffd590b1485b3caadc19b22e6379c733355108f107a430458cdf3407ab6"));
    //assert(genesis.hashMerkleRoot == uint256S("0xe0028eb9648db56b1ac77cf090b99048a8007e2bb64b68f092c03c7f56a662c7"));

Alternatywnie moglibyśmi sobie raz wyliczyć te wartości, wypisać i zakodować je w kodzie.

Wszędzie indziej, gdzie wartość hash pierwszego bloku była zakodowana zmieniamy na consensus.hashGenesisBlock.

Tutaj zapomniałem tego zrobić, ale będzie to poprawione w następnym commit'cie:

    // By default assume that the signatures in ancestors of this block are valid.
    consensus.defaultAssumeValid = uint256S("0x00000ffd590b1485b3caadc19b22e6379c733355108f107a430458cdf3407ab6");

### Aktywacja InstantSend przez jeden masternode

<https://github.com/panmareksadowski/dash/commit/e0078076c04026f355db754f2df2cda51c4320d4>

W pliku [instantx.h](https://github.com/panmareksadowski/dash/blob/master/src/instantx.h) znajdują się stałe <code>SIGNATURES_REQUIRED</code> i <code>SIGNATURES_TOTAL</code>.

Aby wejścia transakcji InstantSend zostały zablokowane(a tym samym transakcja się powiodła) musi za tym zagłosować przynajmniej <code>SIGNATURES_REQUIRED</code>(domyślnie 6) masternode'ów z posród <code>SIGNATURES_TOTAL</code>(domyślnie 10) masternode'w. <code>SIGNATURES_TOTAL</code>(domyślnie 10) jest wybieranych z pośród wszystkich aktywnych masternode'ów.

Aby przetestować czy <code>InstantSend</code> działa w utworzonej przeze mnie sieci zawierającej tylko jednego masternode'a zmieniłem wartość <code>SIGNATURES_REQUIRED</code> na 1. Oczywiście w normalnie działającej sieci z wieloma działającymi masternod'ami ta wartość powinna być równa co najmniej 6 dla bezpieczeństwa transakcji.

#### Drugi serwer

<https://github.com/panmareksadowski/dash/commit/be81b9536b8a0abd00206f8aaee9755d3930ebd7> 

Dodanie adresu drugiego serwera. 

#### Zmiana epoch time genesis block

<https://github.com/panmareksadowski/dash/commit/ff90ed08af6285d5e334f611ab04c87da725d897>

Po usunięciu wykopanych już bloków i po odpaleniu całej sieci od nowa po kilku godzinach przerwy okazało się, że node'y nie chcą kopać nowych bloków.

Przyczyną było to, że jeśli ostatnio znany blok jest starszy niż wartość <code>nMaxTipAge</code> nowe bloki nie są kopane zakładając, że jeszcze nie cały dostępny łańcuch został pobrany. Chyba, że <code>fMiningRequiresPeers</code> jest ustawione na <code>false</code>, wtedy cały czas node próbuje kopać niezależnie od stanu synchronizacji jego łańcucha z innymi node'ami.

Aby naprawić problem zaktualizowałem czas genesis block i zmieniłem wartość <code>nMaxTipAge</code> na 24h(jeśli przez taki czas nie zostanie wykopany żaden blok to sieć "stanie"). Ewentualnie można ustawić flagę <code>fMiningRequiresPeers</code> na <code>false</code>(być może można to zrobić także używając odpowiedniej komendy bez potrzeby zmian w kodzie).

# Uruchamianie Node i Masternode

Identycznie jak w zwykłej sieci Dash. Aby odpalić Node wystarczy uruchomić demona dashd.

Masternode -> <https://dashpay.atlassian.net/wiki/spaces/DOC/pages/113934340/Masternode+Setup>

# Transakcje

Obydwie transakcje zostały wysłane na adres <code>XhpwLvKeSKQHYfFKxbQTDt9gdKf9kFUTNS</code>. Aby je zobaczyć trzeba chyba zaimportować ten adres do portfela komendą importaddress.

Przynajmniej komenda getransaction nie zadziała bez wcześniejszego importu tego adresu.

### Normalna

txid: <code>12c5cd806c9c092813858a66e72ee83bdc074470c275d100fb3619f413815309</code>

    {
      "amount": 0.00000000,
      "confirmations": 11,
      "instantlock": false,
      "blockhash": "00000e813206918c0ce14760fc970aab3c60ac7ac4ce042811bc11c4bc2c6857",
      "blockindex": 1,
      "blocktime": 1511858195,
      "txid": "12c5cd806c9c092813858a66e72ee83bdc074470c275d100fb3619f413815309",
      "walletconflicts": [
      ],
      "time": 1511858195,
      "timereceived": 1511858302,
      "bip125-replaceable": "no",
      "details": [
      ],
      "hex": "01000000017c17328ed040242a5a0792407937c907ad893d336d87ef3bc21a5dc9c31212e500000000484730440220061a739cfcf41db6f1e435cdaf870c322e3b2e65e82034a20699edf0f8985fe3022051f1512a71045ae8c1b7dadf6d0d7c5a9ab08fabd6479d3603dbafa1abe44e4f01feffffff02002f6859000000001976a9144e50cf1702018131bfb78ebb0a552801df4dc5ce88ac1436d34a0b0000001976a914c085475dc6d49ab6be293b9b811c2efb8cabab9d88ac72010000"
    }



### Instantsend

txid: <code>89d333e5bfe489b116719f5b8cfe65b1b5792cc4c83602bbf07da1d67091d4a0</code>

    {
      "amount": 0.00000000,
      "confirmations": 4,
      "instantlock": true,
      "blockhash": "00000022c6e40c6cb47743d6e5c4e419b2082655077af5d8a0c4b52c9fac02eb",
      "blockindex": 1,
      "blocktime": 1511858764,
      "txid": "89d333e5bfe489b116719f5b8cfe65b1b5792cc4c83602bbf07da1d67091d4a0",
      "walletconflicts": [
      ],
      "time": 1511858763,
      "timereceived": 1511858763,
      "bip125-replaceable": "no",
      "details": [
      ],
      "hex": "01000000018dffe592856ac4d7a47e91addd6019f73ef8f231206b15ccefd271a9ea0306000000000049483045022100afc1cc827416413adc79f96420ea756952c381362ea9cdfb9ccb3d8276d24a920220302b34d4335de6697c7bfdb2d6e51631a543b158398a1871c64d6db0b203058701feffffff0200b33f71000000001976a9144e50cf1702018131bfb78ebb0a552801df4dc5ce88ac60c054de060000001976a914b5117debcf36fedf2b1a60c8243027ae51f3012f88accc010000"
    }

Pole <code>instantlock</code> mamy ustawione na <code>true</code>. Także widzimy, że transakcja faktycznie została wykonana jako instant. Faktycznie też środki zostały przelane od razu.

## Branch myNetFlag

<https://github.com/panmareksadowski/dash/tree/myNetFlag>

Na oddzielnym branchu dodałem możliwość uruchamiania zarówno demona(<code>dashd</code>) jak i klienta rpc(<code>dash-cli</code>) z flagą <code>-mynet</code>. Uruchomienie programów bez tej flagi odpali je dla głównej sieci dash <code>mainnet</code>. Zmieniłem też port rpc na <code>18998</code> dla mojej sieci. W ten sposób można bez problemu działać jednocześnie na głównej sieci i na tej utworzonej przeze mnie. Jeśli chcemy teraz odpalić maternode'a koniecznie musimy dodać w pliku <code>dash.conf</code> <code>rpcport=18998</code>, aby sentinel wiedział na jakim porcie działa rpc.
