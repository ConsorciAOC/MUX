# **DI – MUX V3**
Registre unificat d’assentaments d’entrada i sortida


**Índex**

- [1. Introducció](#1)
   - [1.1 Descripció general del servei MUXv3](#1.1)
- [2 Autenticació](#2)
   - [2.1 Connectivitat](#2.1)
   - [2.2 Autenticació i autorització](#2.2)
- [3 Transacció de MUXv3](#3)
   - [3.1 ntroducció](#3.1)
   - [3.2 Blocs que formen la transacció](#3.2)
- [4 Operacions](#4)
   - [4.1 Operacions disponibles](#4.1)
   - [4.2 Entorns](#4.2)
- [5 Integració de Registres Electrònics](#5)
   - [5.1 Solució proposada](#5.1)
   - [5.2 Autenticació de MUXv3 davant el registre electrònic](#5.2)
   - [5.3 Missatge de petició](#5.3)
   - [5.4 Missatge de resposta](#5.4)
   - [5.5 Mòdul d'exemple](#5.5)

# 1 Introducció <a name="1"></a>

Aquest document detalla la missatgeria, operacions i fluxos que cal seguir per a que les eines de registre externes s’integrin amb el servei MUXv3.

## 1.1 Descripció general del servei MUXv3 <a name="1.1"></a>

MUXv3 és un servei destinat a fer d’intermediari entre serveis/aplicacions que necessiten realitzar assentaments i els registres electrònics dels diferents organismes públics. MUXv3 trasllada les dades informades per les aplicacions cap als registres electrònics adients, realitzant transformacions de missatgeria si escau, i usant les dades de la pròpia petició per a determinar cap a quin registre ha de dirigir la petició. 

MUXv3 manté el mateix enfocament que la versió anterior (V2), tot incorporant funcionalitats que venen a cobrir les mancances detectades durant els anys de funcionament de la versió actual. Els punts més rellevants d’aquesta evolució son:


- **Informació ampliada al missatge de dades de l'assentament** [1](#111): Les dades a la missatgeria de la versió actual s’han demostrat insuficients i en la majoria de casos no permeten als registres telemàtics classificar de manera automàtica els assentaments generats, forçant la intervenció humana per a realitzar aquesta classificació. La missatgeria de MUX V3 incorpora molta més informació dirigida a facilitar tant la classificació com la comprensió de l’objecte de l’assentament.

- **Lliurament fiable dels fitxers adjunts a l'assentament** : La versió 2 de MUX no imposa cap mecanisme que garanteixi que els documents als quals l’aplicació que fa el registre fa referència, dins de  l’assentament, siguin accessibles per al registre electrònic. MUXv3 fa us del servei DESA’L del Consorci AOC per a desar els documents adjunts, i verifica a l’hora de crear l’assentament que aquests existeixen i que son accessibles.

- **Bústia d'assentaments realitzats al registre auxiliar** : La versió 2 de MUX només posa a disposició dels registres electrònics un servei genèric de cerca i consulta dels assentaments realitzats via MUX per a revisar si hi ha assentaments fets al seu registre auxiliar. Aquest mecanisme, que ha resultat ser molt poc eficient, ha estat reemplaçat a MUXv3 per un servei de bústies d’assentaments realitzats al registre auxiliar. Cada registre podrà recuperar de la seva bústia les transaccions que hagin rebut l’assentament del registre auxiliar.

- **Operació de consolidació d'assentaments** : MUXv3 inclou una operació per a que el registre telemàtic titular pugui realitzar la consolidació d’un assentament fet al registre auxiliar. La consolidació ajudarà els serveis del Consorci AOC ja que podrà mostrar dades més precises.

- **Adjunció dels documents de rebut o justificants** : MUXv3 permet als serveis integradors adjuntar els documents generats a mode de rebut amb el resum i número d’assentament per tal de que els registres telemàtics els puguin obtenir si els necessiten. Al igual que amb els fitxers adjunts, MUXv3 garanteix que el document del rebut es trobi disponible [2](#222).

----

**111** <a name="111"></a> Cal fer palès que per a que la informació que conté la missatgeria del MUX V3 sigui útil al registre telemàtic, aquest haurà d’haver realitzat la implementació del frontal de referència V3, ja que la versió actual del registre unificat (o de referència) no permet traslladar tota aquesta informació.

***222*** <a name="222"></a> MUXv3 verificarà que en el moment d’adjuntar els documents, aquests es troben a DESA’L i l’accés hi està permès. No obstant, DESA’L por ser configurat per a permetre que un usuari o servei esborri els seus documents. L’aplicació integradora haurà de tindre en compte això i gestionar els documents de manera que el registre electrònic els pugui arribar a descarregar o fer seus.

----

# 2 Autenticació <a name="2"></a>

## 2.1 Connectivitat <a name="2.1"></a>

Per a protegir el servei davant de consums no autoritzats s'ha activat la restricció d'accés per adreces IP.

Com a primer pas per a integrar-se serà necessari verificar la connectivitat i, en cas d'obtenir un error de tipus **403 Forbidden** caldrà contactar amb la unitat de Suport a Integradors del Consorci AOC per tal d'autoritzar l'accés al servei.

## 2.2 Autenticació i autorització <a name="2.2"></a>

L'autenticació a MUXv3 es realitza mitjançant l'ús de _tokens JWT_. Aquests _tokens_ contenen tota la informació necessària per a realitzar les tasques d'autenticació i autorització del peticionari.

Un _token_ és simplement un objecte en notació JSON que conté una sèrie de camps definits a l'estàndard _RFC 7519_. Els camps que MUXv3 requereix son els següents:

- ___iss___: Aquest camp (_Issuer_) estableix l'emissor del _token_. Dins d'aquest camp s'haurà d'informar el literal que identifica l'integrador. Aquest literal serà assignat per la unitat de Suport a Integradors del Consorci AOC. Com a exemples de valors per a aquest camp tenim 'eNOTUM', 'eVALISA', 'OVER', 'EACAT', 'SIFECAT'...
- ___sub___: Aquest camp (_Subject_) estableix l'organisme en nom de qui es fa la petició. Aquest camp pot contenir l'identificador INE10 o DIR3 de l'organisme, per exemple '9821920002'.
- ___aud___: Aquest camp (_Audience_) estableix el servei al qual va dirigit el _token_. D'aquesta manera s'evita l'ús indegut de _tokens_ generats per a altres serveis. Aquest camp **haurà de contenir el literal 'mux_v3'**.
- ___nbf___: Aquest camp (_Not Before_) conté la data, expressada en format _epoch_ amb precisió de segons, a partir de la qual el _token_ entrarà en vigència. Aquest mecanisme es preveu per a poder emetre _tokens_ que començaran a ser vàlids en un instant futur. Normalment però, aquest camp contindrà la data actual.
- ___iat___: Aquest camp (_Issued At_) conté la data d'emissió, en format _epoch_ amb precisió de segons, del token.
- ___exp___: Aquest camp (_Expires At_) conté la data d'expiració, en format _epoch_ amb precisió de segons, del token. Tot i no ser un camp obligatori per part de MUXv3, és recomanable emetre els _tokens_ amb una vigència d'uns pocs segons per a evitar-ne un ús indegut en cas de robatori d'aquests.
- ___user___: Aquest camp és opcional i està destinat a contenir l'identificador de l'usuari final que origina la petició. El valor d'aquest camp s'usarà únicament amb finalitats de traçabilitat i auditoria.
- ___name___: Aquest camp és opcional i està destinat a contenir el nom de l'usuari final que origina la petició. El valor d'aquest camp s'usarà únicament amb finalitats de traçabilitat i auditoria.

Un cop emplenat el token aquest s'haurà de signar amb l'algorisme HMAC-SHA256, usant una clau secreta que serà emesa per la unitat de Suport a Integradors del Consorci AOC. Aquesta clau haurà de ser custodiada de forma segura per part de l'aplicació integradora i **no haurà de viatjar mai** dins d'una petició o capçalera HTTP.

La clau secreta estarà formada per una cadena de text representant un conjunt aleatori de bytes codificat en notació Base64 _URL-safe_. Per exemple:

```
pz7Qwh-_fOIM0gPGOPmkPdmLJ-F0LKcUwDMlmZTVyUW4qe_0FBXm7TDf
```

A continuació es mostra un exemple de _token JWT_ en clar, abans de codificar-se en _Base64_.

```json
{
"alg": "HS256",
 "typ": "JWT"
}
.
{
      "iss": "eVALISA",
      "sub": "9821920002",
      "user": "11111111H",
      "name": "Josep Josep Josep",
      "iat": 1516239022,
      "nbf": 1516239022,
      "exp": 1516239042
}
.
3oGx4zGJy7YfYTJlBiPNBPCUGrC8oNlKQaGHSO4D5M0
```


El primer segment del _token_ conté l'algorisme de signatura emprat per a protegir el _token_, així com el tipus de _token_. Per a MUXv3 **només es suporta l'algorisme** ___'HS256'___ **i el tipus de** ___token___ ___'JWT'___ . Dins del segon segment s'hi troba les dades d'autenticació i autorització comentades. Al tercer segment hi trobem la signatura del _token_, necessària per a verificar que no ha estat alterat, així com la seva autenticitat.

Un cop generat el _token_, aquest s'inclourà a la capçalera HTTP '_Authorization_' de la següent manera:

```json

Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJBUFAwMDAxIiwic3ViIjoiOTgyMTkyMDAwMiIsInVzZXIiOiIxMTExMTExMUgiLCJuYW1lIjoiSm9zZXAgSm9zZXAgSm9zZXAiLCJpYXQiOjE1MTYyMzkwMjIsIm5iZiI6MTUxNjIzOTAyMiwiZXhwIjoxNTE2MjM5MDQyfQ.3oGx4zGJy7YfYTJlBiPNBPCUGrC8oNlKQaGHSO4D5M0

```

A l'exemple de capçalera HTTP amb el _token JWT_ es mostren els diferents segments del _token_ pintats de diferent color. Com es pot observar, els segments està separat per un punt '.'.

Per a més informació sobre els ___tokens___ ___JWT___, eines online per a familiaritzar-se amb el seu ús i, fins i tot, llibreries per a la seva generació i tractament en diversos llenguatges de programació, podeu visitar la següent URL:

[https://jwt.io/](https://jwt.io/)

# 3 Transacció de MUXv3 <a name="3"></a>

## 3.1 Introducció <a name="3.1"></a>

Una característica important de MUXv3 és que usa el concepte de "transacció". És a dir, l'integrador crea en un primer pas el que ve a ser el context d'execució sobre el qual es llançarà la resta d'operacions.

![1](captures/1.png)

Una transacció es configura amb dos paràmetres: El _tipus_ i el temps de vida o _TTL_.

El _tipus_ de transacció es correspon amb els tipus d'assentament que MUXv3 haurà de generar contra els registres electrònics en el moment d'executar la transacció. Així, es suporta tres tipus de transacció:

- _Transacció d'entrada_ per a crear un assentament d'entrada.
- _Transacció de sortida_ per a crear un assentament de sortida.
- _Transacció dual o interadministrativa_ per a crear, amb les mateixes dades, l'assentament de sortida i el contra-assentament d'entrada.

El temps de vida o _TTL_ d'una transacció indica durant quant de temps MUXv3 haurà de permetre realitzar operacions sobre la transacció. La finalitat d'aquest paràmetre és permetre l'alliberament dels recursos que es reserven per a poder executar cada transacció de manera eficient. Normalment, una transacció tindrà un temps de vida d'uns pocs minuts (de 1 a 10 minuts) tot i que es pot arribar a configurar fins a 60 minuts.

Un cop creada la transacció ja es podrà executar, sense cap ordre en particular, les diferents operacions disponibles per a l'establiment de les dades que conformen la transacció: Les _dades d'ofici_, els _documents adjunts_ i els _registres electrònics_ d'origen i/o destí.

Un cop establertes les dades es podrà procedir a executar la transacció, moment en el qual MUXv3 invocarà els serveis de registre electrònic adients i crearà els assentaments.

La operació d'execució de la transacció és _idempotent_, és a dir, si s'ha obtingut un número d'assentament i es torna a executar la transacció, no es tornarà a invocar el registre, si no que es retornarà el número d'assentament ja obtingut a la execució anterior. Aquest comportament cobra especial sentit, sobretot, en el cas de les transaccions _duals o interadministratives_ en les que es pot donar el cas en que un dels assentaments sí que es realitza correctament però el seu contra-assentament no, pel que pot ser necessari rellançar la execució. S'il·lustra a continuació aquest comportament:

##### Transacció d'entrada o sortida:

![2](captures/2.png)

- El primer cop que s'executa la transacció s'invoca el registre electrònic i s'obté un número d'assentament que es desa dins de la transacció de MUXv3.
- El segon cop que s'executa la transacció es retorna directament les dades de l'assentament obtingut durant la primera execució i el registre electrònic no és invocat.

##### Transacció dual:

![3](captures/3.png)


- El primer cop que s'executa la transacció s'invoquen tant el registre electrònic de sortida com el registre electrònic d'entrada. En aquest exemple, l'assentament d'entrada es realitza correctament però el de sortida no. El número d'assentament obtingut es desa dins de la transacció de MUXv3.

- El segon cop que s'executa la transacció es torna a invocar el registre de sortida, ja que la invocació anterior havia fallat. En canvi, el registre d'entrada no serà invocat ja que a la transacció ja consta un número d'assentament vàlid. A l'exemple que es mostra al gràfic, s'obté un número d'assentament de sortida i es retorna a l'integrador els dos assentaments obtinguts com a resultat de les dues execucions.

Totes les dades d'una transacció poden ser modificades fins que s'aconsegueixi realitzar al menys un assentament. Així, si la execució d'una transacció falla degut al valor incorrecte d'un camp, es pot modificar el valor d'aquest camp i tornar a executar la transacció (sempre i quan aquesta no hagi expirat i no hagi estat tancada pel propi integrador).

## 3.2 Blocs que formen la transacció <a name="3.2"></a>

Una transacció està formada, a alt nivell, pels següents blocs:

![4](captures/4.png)

Com es veu al gràfic, els blocs que formen la transacció son:

- ___Atributs bàsics de transacció___ : Conté atributs com l'identificador de la transacció, el seu tipus, el nombre de cops que s'ha executat, data de creació, data de darrera execució i el _TTL_ restant.</li>

- ___Dades d'Ofici___</li>
  - Control
  - Assumpte
  - Procediment
  - Interessats
  - Instància Genèrica
  - Dades Lliures</br>
- ___Registres Electrònics___ </li>
  - Registre Electrònic d'Origen (per a les sortides)
  - Registre Electrònic de Destí (per a les entrades)</br>
- ___Documents Adjunts___</li>
  - Llista dels documents adjunts a la transacció<br>
- ___Dades dels Assentaments___</li>
  - Assentament d'Entrada
    - Registre Principal
    - Registre Auxiliar
    - Dades de Consolidació
  - Assentament de Sortida
    - Registre Principal
    - Registre Auxiliar
    - Dades de Consolidació<br>
- ___Rebuts/Justificants___</li>
  - Rebut de l'assentament d'entrada
  - Rebut de l'assentament de sortida

   
### 3.2.1 Atributs bàsics de la transacció <a name="3.2.1"></a>

A continuació es mostra els atributs bàsics de la transacció:

**idTransaccio** : Identificador de la transacció

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | 0b81-3363-eb40-c004 |

L'identificador de transacció és un número de 64 bits expressat en format hexadecimal, organitzat en quatre grups separats per guions '-'.

Aquest identificador és generat per MUXv3 en el moment de creació d'una nova transacció i haurà de ser conservat per l'aplicació integradora per a poder executar les següents operacions.

**execucions** : Nombre d'execucions

| **tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 1 |

Aquest camp conté el valor del comptador del nombre de vegades que s'ha executat la transacció.

**tipus** : Tipus de transacció

| **tipus** | Enumeració de tipus string |
| --- | --- |
| **valors d'exemple** | E, S, ES |

Aquest camp estableix de quin del tres possibles tipus de transacció es tracta. Els valors possibles son:

|- | - |
|--- | --- |
| **E** | Transacció d'entrada |
|| _Una transacció d'entrada generarà únicament un assentament d'entrada contra el registre electrònic que correspongui._ |
| **S**| Transacció de sortida |
|| _Una transacció de sortida generarà únicament un assentament de sortida contra el registre electrònic que correspongui._ |
| **ES** | Transacció dual o interadministrativa |
|| _Una transacció dual generarà l'assentament de sortida i a continuació el contra-assentament d'entrada contra el parell de registres electrònics que correspongui._ |

**dataCreacio** : Data de creació de la transacció

| **tipus** | Long |
| --- | --- |
| **valor d'exemple** | 1617719479214 |

Aquest camp indica la data, expressada en mil·lisegons des de l'1 de gener de 1970, en la qual es va crear la transacció. El valor donat com a exemple correspondria al _dimarts, 6 d'abril de 2021 a les 16:31:19.214 GMT+2_.

**dataDarreraExecucio** : Data de la darrera execució de la transacció

| **Tipus** | Long |
| --- | --- |
| **valor d'exemple** | 1617719538576 |

Aquest camp indica la data, expressada en mil·lisegons des de l'1 de gener de 1970, en la qual es va executar per darrer cop la transacció. El valor donat com a exemple correspondria al _dimarts, 6 d'abril de 2021 a les 16:32:18.576 GMT+2_.

**TTL** : Temps de vigència restant de la transacció

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 120 |

Aquest camp indica el temps de vida/vigència, expressat en segons, del que encara disposa la transacció per a poder ser executada. El valor d'exemple donat correspondria a un temps de vigència restant de 2 minuts (2x60 segons).

A continuació mostrem un exemple de transacció amb aquests camps:
```jsp
{
      "idTransaccio":"0b81-3366-eb41-c004",
      "execucions":2,
      "tipus":"ES",
      "dataCreacio":1617719479214,
      "dataDarreraExecucio":1617719538576,
      "ttl":10
}
```

    
#### 3.2.2 Bloc _Ofici_ <a name="3.2.2"></a>

En aquest apartat es troba la definició dels diferents blocs que formen el bloc de dades d'ofici. Recordem quins eren aquests blocs:

- Control
- Assumpte
- Procediment
- Interessats
- Instància Genèrica
- Dades Lliures

      
##### 3.2.2.1 Bloc _Control_

Dins d'aquest bloc s'informarà una sèrie de camps dirigits a habilitar la exportació a format SICRES de la transacció MUXv3, de fet, aquest bloc es basa en el bloc amb el mateix nom de la especificació SICRES. A continuació es detalla els camps que el formen:

- **aplicacio** : Identificador del servei o aplicació

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | eNOTUM |

L'identificador d'aplicació és un literal que identifica el servei o aplicació que vol realitzar l'assentament. Per a cada aplicació s'haurà de definir un literal i, un cop decidit, informar sempre el mateix literal. El valor d'aquest camp ha de coincidir amb el camp _iss_ (_issuer_) del token JWT d'autenticació.

- **urlAplicacio** : URL del servei o aplicació

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | https://serveis3.app.aoc.cat/over/tramits?idTramit=3423 |

Aquest camp s'usarà per a informar una URL, pròpia del servei o aplicació que vol realitzar l'assentament, que pugui ser d'interès per al registre electrònic o els seus usuaris. Per exemple, es pot informar la URL al tràmit que ha originat la creació de l'assentament, o bé a una plana que dona informació genèrica relacionada, com una normativa o una circular.

- **observacionsApunt** : Comentari genèric

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | Informació addicional sobre la nova normativa... |

Aquest camp s'usarà per a introduir un text breu a mode de comentari respecte a l'assentament a realitzar.

- **documentacioFisica** : Indicador d'existència de documentació física associada

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | 1, 2, 3 |

Aquest camp indica si la transacció electrònica té associada documentació física addicional o no. Els valors possibles son:

| - | - |
| --- | --- |
| **1** | Documentació física requerida |
|  | _Aquest valor informa el registre electrònic de que la transacció té associada documentació física (p.e. en format paper) necessària per a la realització de l'assentament._ |
| **2** | Documentació física complementària |
|| _Aquest valor informa el registre electrònic de que la transacció té associada documentació física addicional._ |
| **3** | Sense documentació física |
|| _Aquest valor informa el registre electrònic de que la transacció no té associada cap mena de documentació física i que, per tant, l'assentament serà purament electrònic._ |

- **indicadorProva** : Indicador de transacció de proves

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | 0, 1 |

Aquest camp indica al registre electrònic si ha de generar un número d'assentament real (amb totes les implicacions legals i de negoci associades) o si, per contra, es tracta d'una transacció llançada amb la intenció de provar el bon funcionament del circuit, per exemple, durant la correcció d'un error a la integració.

Els valors possibles per a aquest camp es descriuen a continuació:

| - | - |
| --- | --- |
| **0** | Transacció real |
|| _La transacció haurà de ser processada pel registre i generar un assentament vàlid._ |
| **1** | Transacció de proves |
|| _El registre haurà de processar la transacció per a verificar que aquesta és correcta però haurà de generar un número d'assentament fictici._ |

- **usuari** : Usuari que origina la creació de l'assentament

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | 11111111H |

Aquest camp s'usarà per a introduir-hi l'identificador d'usuari que ha originat la creació de l'assentament a través del servei o aplicació integrador.

- **contacteUsuari** : Dada de contacte de l'usuari que origina la creació de l'assentament

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | usuari@evalisa.aoc.cat |

Aquest camp s'usarà per a introduir-hi un text que correspongui a un mitjà de contacte (per exemple una adreça de correu electrònic o un telèfon) amb l'usuari que ha originat la creació de l'assentament a través del servei o aplicació integrador.

- **tipusTransport** : Indicador del mitjà de transport de la documentació física associada

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | 01, 02, 03 , 04, 05, 06, 07 |

Aquest camp indica el tipus de transport emprat per a fer arribar la documentació física associada a l'assentament.

Els valors possibles son:

| - | - |
| --- | --- |
| **01** | Missatgers |
|| _La documentació física associada ha estat enviada per missatger._ |
| **02** | Correu Postal |
|| _La documentació física associada ha estat enviada per correu ordinari._ |
| **03** | Correu Postal Certificat |
|| _La documentació física associada ha estat enviada per correu certificat._ |
| **04** | Burofax |
|| _La documentació física associada ha estat enviada per burofax._ |
| **05** | Entrega en mà |
|| _La documentació física associada ha estat o serà entregada en mà._ |
| **06** | Fax |
|| _La documentació física associada ha estat enviada per Fax._ |
| **07** | Altres |
|| _No s'ha especificat com s'entregarà la documentació física associada._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **numeroTransport** : Numero de seguiment de l'enviament de la documentació física

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | SB00013564889823 |

En cas d'existir l'enviament de documentació física associada, aquest camp indica el número de seguiment proporcionat pel servei de missatgeria o correu emprat.

- **identificadorIntercanvi** : Identificador SICRES d'intercanvi registral

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | A09018933\_21\_00000001 |

Aquest camp ha de ser informat al moment de realitzar un intercanvi registral a través del sistema SIR i s'ha incorporat a la transacció de MUXv3 per a complir amb la especificació SICRES, tot i que no aplicarà en la gran majoria dels casos.

- **numeroAssentamentIntercanvi** : Numero d'assentament d'intercanvi registral

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | E/000123-2021 |

Aquest camp ha de ser informat si es vol traslladar al registre electrònic un número d'assentament previ.

- **tipusAnotacio** : Tipus d'operació de registre

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | 01, 02, 03, 04 |

Aquest camp, que respon a la especificació SICRES, està dirigit al seu ús per part del sistema d'intercanvi registral (SIR) i indica el tipus d'operació que s'està realitzant dins d'aquest sistema.

Els valors que admet aquest camp son:

| - | - |
| --- | --- |
| **01** | Pendent |
|| _Aquest valor indica que el missatge SICRES es troba pendent de processament._ |
| **02** | Enviament |
|| _Aquest valor indica que el missatge SICRES ha estat enviat al seu possible destinatari._ |
| **03** | Reenviament |
|| _Aquest valor indica que el missatge SICRES ha estat reenviat cap a un nou receptor ja que havia estat enviat a algú que no n'era el destinatari final._ |
| **04** | Rebuig |
|| _Aquest valor indica que el missatge no ha estat acceptat pel seu destinatari._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

Per a més informació sobre la especificació SICRES, podeu consultar la documentació al lloc web del Ministerio de Asuntos Económicos y Transfomación Digital.

[https://administracionelectronica.gob.es/ctt/sicres#.YG7M8q\_7SUk](https://administracionelectronica.gob.es/ctt/sicres#.YG7M8q_7SUk)

A continuació es mostra un exemple del bloc 'Control' dins de la transacció:

```json
 ...
      "control":
      {
            "aplicacio":"eNOTUM","urlAplicacio":"https://serveis3.app.aoc.cat/over/tramits?idTramit=3423",
            "observacionsApunt":"Informació addicional sobre la nova normativa...",
            "documentacioFisica":{
                  "clau":"3","descripcio":"Sense documentació física"
            },
            "indicadorProva":{
                  "clau":"0",
                  "descripcio":"Transacció real"
            },
            "usuari":"11111111H",
            "contacteUsuari":"usuari@evalisa.aoc.cat",
            "tipusTransport":{
                  "clau":"01",
                  "descripcio":"Missatger"
            },
            "numeroTransport":"SB00013564889823",
            "identificadorIntercanvi":"A09018933_21_00000001",
            "tipusAnotacio":{
                  "clau":"02",
                  "descripcio":"Enviament"
            }
      }
... 
```


##### 3.2.2.2 Bloc _Assumpte_

Dins d'aquest bloc s'informarà una sèrie de camps dirigits proporcionar informació sobre la finalitat o objecte de l'assentament sol·licitat. A continuació es detalla els camps que el formen:

- **resum** : Descripció de l'assumpte o objecte de l'assentament

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | Tramitació dels ajuts i beques per als menjadors 2020 |

Aquest camp s'usarà per a donar una descripció breu de l'objecte de l'assentament.

- **observacions** : Informació addicional complementària al resum de l'assumpte.

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | Període de tramitació excepcional degut a les restriccions derivades... |

Aquest camp s'usarà per a ampliar la informació respecte a la finalitat o objecte de l'assentament amb un text més llarg.

- **dataPresentacio** : Data de presentació efectiva

| **tipus** | Long |
| --- | --- |
| **valor d'exemple** | 1617719538576 |

Aquest camp indica la data de presentació efectiva, expressada en mil·lisegons des de l'1 de gener de 1970, que haurà d'observar l'aplicació de registre electrònic. El valor donat com a exemple correspondria al _dimarts, 6 d'abril de 2021 a les 16:32:18.576 GMT+2_.

- **codiAssumpteSegonsDesti** : Codi de l'assumpte segons codificació del destí

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | RR000001 |

Aquest camp s'usarà per a indicar el codi corresponent a l'assumpte de l'assentament segons la codificació del destinatari, si l'integrador el coneix.

- **referenciaExterna** : Referència externa

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | REF00001 |

Aquest camp s'usarà per a incloure una referència externa.

- **numeroExpedient** : Número d'expedient associat a l'assentament

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | SH/2021/00034-2 |

Aquest camp s'usarà per a indicar el número d'expedient al qual estarà associat l'assentament. Aquest número d'expedient serà local respecte a l'integrador.

- **numeroExpedientSegonsDesti** : Número d'expedient segons el destinatari

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | SH/2021/0003314 |

Aquest camp s'usarà per a indicar el número d'expedient al qual estarà associat l'assentament per al destinatari d'aquest.

- **codiFamilia** : Codi de família de procediments

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | FAM00345 |

Aquest camp s'usarà per a indicar un codi de família de procediments.

- **nomFamilia** : Descripció de la família de procediments

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | Procediments de sol·licitud de subvencions al món local |

Aquest camp s'usarà per a incorporar la descripció del codi de família de procediments informat al camp 'codiFamilia'.

- **codiProcediment** : Codi identificador del procediment

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | PR001\_0035 |

Aquest camp s'usarà per a identificar el procediment administratiu associat a l'assentament.

- **nomProcediment** : Nom del procediment administratiu

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | Sol·licitud de subvencions i ajuts per als menjadors escolars |

Aquest camp s'usarà per a donar el nom, o una breu descripció, del procediment administratiu associat a l'assentament.

- **codiSIA** : Codi SIA del procediment administratiu

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | SIA0001 |

Aquest camp s'usarà per a informar el codi identificador del procediment administratiu seguin la codificació SIA (Sistema de Información Administrativa).

- **nomSIA** : Nom del procediment administratiu segons SIA

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | Sol·licitud de subvencions i ajuts per als menjadors escolars |

Aquest camp s'usarà per a informar el nom, o donar una petita descripció, del procediment administratiu segons s'hagi especificat a SIA.

- **codiTramit** : Codi identificador del tràmit

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | TR0001 |

Aquest camp s'usarà per a informar l'identificador del tràmit dins del procediment administratiu associat a l'assentament.

- **nomTramit** :

| **tipus** | String |
| --- | --- |
| **valors d'exemple** | Formulari preliminar de sol·licitud d'ajuts per al menjador escolar |

Aquest camp s'usarà per a informar el nom del tràmit, dins del procediment administratiu, associat a l'assentament.

A continuació es mostra un exemple del bloc 'Assumpte dins de la transacció:

```json
...
"assumpte":{
   "resum":"Tramitació dels ajuts i beques per als menjadors 2020",
   "observacions":"Període de tramitació excepcional degut a les restriccions derivades...",
   "dataPresentacio":1617719538576,
   "codiAssumpteSegonsDesti":"RR000001",
   "referenciaExterna":"REF00001",
   "numeroExpedient":"SH/2021/00034-2",
   "numeroExpedientSegonsDesti":"SH/2021/0003314",
   "codiFamilia":"FAM00345",
   "nomFamilia":"Procediments de sol·licitud de subvencions al món local",
   "codiProcediment":"PR001\\_0035",
   "nomProcediment":"Sol·licitud de subvencions i ajuts per als menjadors escolars",
   "codiSIA":"SIA0001",
   "nomSIA":"Sol·licitud de subvencions i ajuts per als menjadors escolars",
   "codiTramit":"TR0001",
   "nomTramit":"Formulari preliminar de sol·licitud d'ajuts per al menjador escolar"
}
...

```

      
#### 3.2.2.3 Bloc _Procediment_

Dins d'aquest bloc s'informarà una sèrie de camps dirigits a donar una informació detallada sobre el procediment administratiu associat a l'assentament. A continuació es detalla els camps que el formen:

- **identificadorResolutor** : Indentificador del resolutor del procediment

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | 11111111H |

Aquest camp s'usarà per a informar el document d'identitat del resolutor del procediment.

- **nomResolutor** : Nom del resolutor del procediment

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | Núria Vergés Martí |

Aquest camp s'usarà per a informar el nom del resolutor del procediment.

- **destinatari** : Tipus de destinatari al qual va dirigit el procediment

| **tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | CI, EM, AP |

Aquest camp s'usarà per a indicar el tipus de destinatari al qual va dirigit el procediment. Els valors possibles son:

| - | - |
| --- | --- |
| **CI** | Ciutadà |
|| _Aquest valor indica que el procediment està dirigit a la ciutadania._ |
| **EM** | Empresa |
|| _Aquest valor indica que el procediment està dirigit al món de l'empresa privada._ |
| **AP** | Administració Pública |
|| _Aquest valor indica que el procediment està dirigit a les administracions públiques._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **preuTasa** : Especifica si el procediment té un cost associat

| **tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | False |

Aquest camp s'usarà per a informar si el procediment té associat un cost a mode de tasa.

- **inici** : Defineix per part que qui s'inicia el procediment

| **tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | DI, DO |

Aquest camp s'usarà per a informar de qui inicia el procediment. Els valors possibles son:



| - | - |
| --- | --- |
| **DI** | Interessat |
|| _Aquest valor indica que el procediment serà iniciat per part de l'interessat._ |
| **DO** | Ofici |
|| _Aquest valor indica que el procediment serà iniciat d'ofici._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **efecteSilenci** : Efecte sobre el procediment del silenci administratiu

| **tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | PO, NE, SN, CA, NA |

Aquest camp s'usarà per a especificar l'efecte que tindrà el silenci administratiu sobre la resolució del procediment. Els valors possibles son:

| - | - |
| --- | --- |
| **PO** | Positiu |
|| _El procediment es resoldrà favorablement._ |
| **NE** | Negatiu |
|| _El procediment es resoldrà desfavorablement._ |
| **SN** | Segons Normativa Aplicable |
|| _El procediment es resoldrà segons la normativa aplicable en el cas._ |
| **CA** | Caducitat |
|| _El procediment caducarà._ |
| **NA** | No té |
|| _No s'especifica._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **tipusProcediment** : Defineix el tipus de procediment

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | EC, IC, IE, EE |

Aquest camp s'usarà per a especificar el tipus de procediment. Els valors possibles son:

| - | - |
| --- | --- |
| **EC** | Extern Comú |
|| _Aquest valor indica que el procediment extern comú_ |
| **IC** | Intern Comú |
|| _Aquest valor indica que el procediment intern comú_ |
| **IE** | Intern específic |
|| _Aquest valor indica que el procediment intern específic_ |
| **EE** | Extern específic |
|| _Aquest valor indica que el procediment extern específic_ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **fiViaAdministrativa** : Fi de la via administrativa

| **tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | True |

Aquest camp indica que el procediment posa fi a la via administrativa.

- **rangNormativa** :

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | RLX, DLL, CIR, RLE, DEC, SEN, ALT, DIR, DLE, RDE, NOC, ORE, DEI, INS, RUE, DFO, ORD, LOR, ACO, RES, ADI, TAI, LFO, LLE |

Aquest camp s'usarà per a informar el rang de la normativa aplicable al procediment.

Els valors possibles per a aquest camp es descriuen a continuació:

| - | - |
| --- | --- |
| **RLX** | Reial Decret Llei |
|| _La normativa en que es recolza el procediment és un Reial decret llei._ |
| **DLL** | Decret Llei |
|| _La normativa en que es recolza el procediment és un Decret llei._ |
| **CIR** | Circular |
|| _La normativa en que es recolza el procediment és una Circular._ |
| **RLE** | Reial Decret Legislatiu |
|| _La normativa en que es recolza el procediment és un Reial decret legislatiu._ |
| **DEC** | Decret |
|| _La normativa en que es recolza el procediment és un Decret._ |
| **SEN** | Sentència |
|| _La normativa en que es recolza el procediment és una Sentència._ |
| **ALT** | Altres |
|| _Normativa indeterminada._ |
| **DIR** | Directiva |
|| _La normativa en que es recolza el procediment és una Directiva._ |
| **DLE** | Decret Legislatiu |
|| _La normativa en que es recolza el procediment és un Decret legislatiu._ |
| **RDE** | Reial Decret |
|| _La normativa en que es recolza el procediment és un Reial decret._ |
| **NOC** | Norma Constitucional |
|| _La normativa en que es recolza el procediment és una Norma constitucional._ |
| **ORE** | Ordenança |
|| _La normativa en que es recolza el procediment és una Ordenança._ |
| **DEI** | Decisió |
|| _La normativa en que es recolza el procediment és una Decisió._ |
| **INS** | Instrucció |
|| _La normativa en que es recolza el procediment és una Instrucció._ |
| **RUE** | Reglament UE |
|| _La normativa en que es recolza el procediment és un Reglament de la UE._ |
| **DFO** | Decret Foral |
|| _La normativa en que es recolza el procediment és un Decret foral._ |
| **ORD** | Ordre |
|| _La normativa en que es recolza el procediment és una Ordre._ |
| **LOR** | Llei Orgànica |
|| _La normativa en que es recolza el procediment és una Llei orgànica._ |
| **ACO** | Acord |
|| _La normativa en que es recolza el procediment és un Acord._ |
| **RES** | Resolució |
|| _La normativa en que es recolza el procediment és una Resolució._ |
| **ADI** | Altres Disposicions |
|| _La normativa en que es recolza el procediment és una Altra disposició._ |
| **TAI** | Tractat i Acord Internacional |
|| _La normativa en que es recolza el procediment és un Tractat i acord internacional._ |
| **LFO** | Llei foral |
|| _La normativa en que es recolza el procediment és una Llei foral._ |
| **LLE** | Llei |
|| _La normativa en que es recolza el procediment és una Llei._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **numeroDisposicio** : Numero de disposició

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | 00000001 |

Aquest camp informa el número de disposició associat al procediment.

- **titolNormativa** :Títol de la normativa

| **tipus** | String |
| --- | --- |
| **valor d'exemple** | Normativa aplicable a.... |

Aquest camp informa el títol de la normativa associat al procediment.

- **materia** : Matèria

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | BAP, RAA, TII, TSI, SSI, SAL, COM, PIC, EIN, RAD, TIT, EPE, TOL, EMP, JUS, EST, CUL, AEX, AFA, CIN, MAB, ARP, CXN, EFO, IAT, CON, PEN, INT, VIU, PCS, ESP, TSS |

Aquest camp s'usarà per a indicar la matèria associada al procediment. Els valors possibles per a aquest camp es descriuen a continuació:

| **BAP** | Beques, Ajuts i Premis |
| --- | --- |
| **RAA** | Relacions entre Administracions Públiques |
| **TII** | Tecnologia, Investigació i Innovació |
| **TSI** | Telecomunicacions i Societat de la Informació |
| **SSI** | Serveis Socials i Igualtat |
| **SAL** | Salut |
| **COM** | Comunicació |
| **PIC** | Participació i Iniciativa Ciutadana |
| **EIN** | Energia i Industria |
| **RAD** | Relacions amb la Administració |
| **TIT** | Trànsit i Transports |
| **EPE** | Economia i Patrimoni de l'Estat |
| **TOL** | Turisme, Oci i Temps lleure |
| **EMP** | Empreses |
| **JUS** | Justícia |
| **EST** | Estadístiques |
| **CUL** | Cultura |
| **AEX** | Acció Exterior |
| **AFA** | Associacions, Fundacions i Altres Entitats |
| **CIN** | Ciutadania i Nacionalitat |
| **MAB** | Medi Ambient |
| **ARP** | Agricultura, Ramaderia, Pesca y Alimentació |
| **EFO** | Educació i Formació |
| **IAT** | Impostos i altres tributs |
| **CON** | Consum |
| **PEN** | Pensions |
| **INT** | Ingressos no Tributaris |
| **VIU** | Vivenda i Urbanisme |
| **PCS** | Protecció Civil, Seguretat Ciutadana i Defensa Nacional |
| **ESP** | Esports |
| **TSS** | Treball i Seguretat Social |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **canalAcces** : Canal d'accés

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | TEL, ELE, PRE, CPO |

Aquest camp s'usarà per a.

Els valors possibles per a aquest camp es descriuen a continuació:

| - | - |
| --- | --- |
| **TEL** | Telefònic |
|| _Aquest valor indica que es suportarà el canal d'accés telefònic per al procediment._ |
| **ELE** | Electrònic |
|| _Aquest valor indica que es suportarà el canal d'accés electrònic per al procediment._ |
| **PRE** | Presencial |
|| _Aquest valor indica que es suportarà el canal d'accés presencial per al procediment._ |
| **CPO** | Correu postal |
|| _Aquest valor indica que es suportarà el canal d'accés per correu postal per al procediment._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **requisitsIdentificacio** : Requeriments d'Identificació del procediment

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | CEL, NRE, EPC, CLV, DNI, USC |

Aquest camp s'usarà per a indicar el requeriment d'identificació necessari per a accedir al procediment.

Els valors possibles per a aquest camp es descriuen a continuació:

| - | - |
| --- | --- |
| **CEL** | Certificat electrònic |
|| _L'accés al procediment requerirà la presentació d'un certificat electrònic._ |
| **NRE** | No requereix |
|| _El procediment no requereix cap tipus d'identificació._ |
| **EPC** | Entrega Presencial de Contrasenya |
|| _L'accés al procediment requerirà una contrasenya entregada presencialment._ |
| **CLV** | ClavePIN |
|| _El procediment requerirà identificació mitjançant @ClavePIN._ |
| **DNI** | DNIe |
|| _El procediment requerirà identificació mitjançant DNIe._ |
| **USC** | Usuari i Contrasenya |
|| _El procediment requerirà la identificació mitjançant usuari i contrasenya._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **urlAccesFormulari** : URL d'accés al formulari

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | https://www.eacat.cat/tramits?ens=9821920002&id\_tramit=1367 |

Aquest camp s'usarà per a indicar la URL mitjançant la qual es pot accedir al procediment.

- **terminiResolucio** : Termini de resolució del procediment

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Aquest procediment es resoldrà en un termini màxim de 15 dies laborables. |

Aquest camp s'usarà per a indicar amb un breu text el termini de resolució del procediment.

- **viaNotificacio** : Indica la via de notificació a l'interessat

| **tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | CSU, POS |

Aquest camp s'usarà per a indicar la via de notificació que s'usarà per a avisar l'interessat.

Els valors possibles per a aquest camp es descriuen a continuació:

| - | - |
| --- | --- |
| **CSU** | Compareixença a Seu |
| | _L'interessat haurà de comparèixer a la seu per a obtenir la notificació._ |
| **POS** | Via Postal |
| |_La notificació serà enviada mitjançant l'enviament postal._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

A continuació es mostra un exemple del bloc 'Procediment' dins de la transacció:

```json
...
"procediment":{
   "identificadorResolutor":"11111111H",
   "nomResolutor":"Núria Parera Martí",
   "destinatari":{
      "clau":"CI",
      "descripcio":"Ciutadà"
   },
   "preuTasa":true,
   "inici":{
      "clau":"DO",
      "descripcio":"D'Ofici"
   },
   "efecteSilenci":{
      "clau":"SN",
      "descripcio":"Segons Normativa Aplicable"
   },
   "tipusProcediment":{
      "clau":"EC",
      "descripcio":"Extern Comú"
   },
   "fiViaAdministrativa":false,
   "rangNormativa"{
      "clau":"RUE",
      "descripcio":"Reglament UE"
   },
   "numeroDisposicio":"1223",
   "titolNormativa":"Normativa regulació",
   "materia"{
      "clau":"EIN",
      "descripcio":"Energia i Indústria"
   },
   "canalAcces"{
      "clau":"ELE",
      "descripcio":"Electrònic"
   },
   "requisitsIdentificacio"{
      "clau":"CEL",
      "descripcio":"Certificat electrònic"
   },
   "urlAccesFormulari":"https://www.eacat.cat/9821920002?tramit=1356&familia\\_tramit=33",
   "terminiResolucio":"15 dies naturals des de la presentació",
   "viaNotificacio"{
      "clau":"POS",
      "descripcio":"Via Postal"
   }
}
...
```

     
#### 3.2.2.4 Bloc _InstanciaGenerica_

Dins d'aquest bloc s'informarà una sèrie de camps dirigits a recollir els camps propis d'una instància genèrica. A continuació es detalla els camps que el formen:

- **exposa** : Aquest camp recull l'exposició que fa l'interessat dels fets que el porten a realitzar la sol·licitud.

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Les rajoles de les aceres del meu carrer es troben en molt mal estat... |

Aquest camp s'usarà per a recollir a mode de breu text, els fets que motiven la sol·licitud.

- **solicita** : Aquest camp recull la sol·licitud que l'interessat fa a tenor dels fets exposats.

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Sol·licito que les aceres siguin reparades quan abans millor per a... |

Aquest camp s'usarà per a recollir el text amb aquelles coses que l'interessat vol sol·licitar.

A continuació es mostra un exemple del bloc 'InstanciaGenerica' dins de la transacció:

```json
...
"instanciaGenerica":{
   "exposa":"Les rajoles de les aceres del meu carrer es troben en molt mal estat...",
   "solicita":"Sol·licito que les aceres siguin reparades quan abans millor per a..."
}
...

 ```

      
#### 3.2.2.5 Bloc _Interessats_

Dins d'aquest bloc s'informarà una llista amb les dades dels interessats:

- **interessats** : Aquest camp és una llista que recull les dades dels interessats.

| **Tipus** | Llista |
| --- | --- |
| **valors** | [{_interessat_}, {_interessat_}, {_interessat_}] |

Cadascun dels elements d'aquesta llista contindrà els següents camps:

- **tipusIdentificador** : Aquest camp recull el tipus d'identificador de l'interessat.

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | PAS, NIF, VAT, CIF, DNI, NIE |

Aquest s'informarà amb el tipus de document d'identitat de l'interessat. Els valors possibles per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **PAS** | Passaport |
|| _El número de document d'identificació és un passaport._ |
| **NIF** | Numero d'Identificació Fiscal |
|| _El número de document d'identificació és un NIF._ |
| **VAT** | Número VAT |
|| _El número de document d'identificació és un identificador fiscal de la UE._ |
| **CIF** | Codi d'Identificació Fiscal |
|| _El número de document d'identificació és un CIF._ |
| **DNI** | Document Nacional d'Identitat |
|| _El número de document d'identificació és un DNI._ |
| **NIE** | Numero d'Identificació de l'Estranger |
|| _El número de document d'identificació és un NIE._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **identificador** : Document d'identificació de l'interessat.

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 11111111H, S0811001G |

Aquest camp s'usarà per a recollir el número de document d'identitat de l'interessat.

- **informacioIdentificador** : Informació ampliada sobre l'identificador.

| **Tipus** | Bloc |
| --- | --- |
| **valor d'exemple** | Veure punt 3.2.2.5.1 |

Aquest bloc, generat pel propi MUXv3, conté una sèrie de camps determinats mitjançant un petit anàlisi del número de document d'identitat de l'interessat i del tipus de document informats pel integrador.

- **dir3personaJuridica** : Identificador de Persona Jurídica amb format DIR3

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | ES11111111H0123 |

Aquest camp s'usarà per a informar l'identificador de persona jurídica amb format DIR3.

- **raoSocial** : Raó social de la organització

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Comercial Fictícia, S.L. |

Aquest camp s'usarà per a informar la raó social de l'interessat.

- **nomSentit** : Nom sentit de l'interessat

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Pep |

Aquest camp s'usarà per a informar el nom sentit, o aquell nom pel qual l'interessat es fa anomenar, i que no es correspon amb el nom present al seu document d'identitat.

Alguns exemples de nom sentit podrien ser:

      - Josep -\> Pep
      - Jordi -\> Yoyi
      - Úrsula -\> Uku

- **nom** : Nom de l'interessat

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Josep Lluís |

Aquest camp s'usarà per a informar el nom formal de l'interessat, tal i com apareix al seu document d'identitat.

- **cognom1** : Cognom primer de l'interessat

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Fernández |

Aquest camp s'usarà per a informar el primer cognom de l'interessat, tal i com apareix al seu document d'identitat.

- **cognom2** : Cognom segon de l'interessat

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Noguera |

Aquest camp s'usarà per a informar el segon cognom de l'interessat, tal i com apareix al seu document d'identitat.

- **genere** : Aquest camp recull el gènere de l'interessat.

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | H, D, NB |

A aquest camp s'hi informarà el gènere de l'interessat. Els valors possibles per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **H** | Home |
|| _El gènere de l'interessat és masculí._ |
| **D** | Dona |
|| _El gènere de l'interessat és femení._ |
| **NB** | No Binari |
|| _El gènere de l'interessat no es defineix com a masculí ni femení._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **sexe** : Aquest camp recull el sexe de l'interessat.

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | M, F, Home, Dona |

A aquest camp s'hi informarà el sexe biològic de l'interessat. Tot i ser semblants, no s'ha de confondre aquest camp amb el camp gènere.

- **codiMunicipi** : Codi INE6 del municipi de residència de l'interessat.

| **Tipus** | String (6 caràcters) |
| --- | --- |
| **valors d'exemple** | 080193 |

Aquest camp contindrà el codi INE6 del municipi de residència de l'interessat. Aquest codi es construeix de la següent manera:

[codiProvincia][codiLocalitat][digitControl]

Aquests codis es poden descarregar a mode de full d'Excel a la web de l'INE a:

[https://www.ine.es/daco/daco42/codmun/21codmun.xlsx](https://www.ine.es/daco/daco42/codmun/21codmun.xlsx)

- **municipi** : Nom del municipi de residència de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Barcelona |

Aquest camp contindrà el nom del municipi de residència de l'interessat.

- **codiProvincia** : Codi de província de residència de l'interessat

| **Tipus** | String (2 caràcters) |
| --- | --- |
| **valors d'exemple** | 08, 25, 17, 43 |

Aquest camp contindrà el codi de província de residència de l'interessat.

Aquests codis es poden descarregar a mode de full d'Excel a la web de l'INE a:

[https://www.ine.es/daco/daco42/codmun/21codmun.xlsx](https://www.ine.es/daco/daco42/codmun/21codmun.xlsx)

- **provincia** : Nom de la província de residència de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Tarragona |

Aquest camp contindrà el nom de la província de residència de l'interessat.

- **codiComunitatAutonoma** : Codi de la comunitat autònoma de residència de l'interessat

| **Tipus** | String (2 caràcters) |
| --- | --- |
| **valors d'exemple** | 09 |

Aquest camp contindrà el codi de la comunitat autònoma de residència de l'interessat.

Els codis i noms de les comunitats autònomes es poden consultar a la web de l'INE a:

[https://www.ine.es/daco/daco42/codmun/cod\_ccaa.htm](https://www.ine.es/daco/daco42/codmun/cod_ccaa.htm)

- **comunitatAutonoma** : Nom de la comunitat autònoma de residència de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Catalunya |

Aquest camp contindrà el nom de la comunitat autònoma de residència de l'interessat.

Els codis i noms de les comunitats autònomes es poden consultar a la web de l'INE a:

[https://www.ine.es/daco/daco42/codmun/cod\_ccaa.htm](https://www.ine.es/daco/daco42/codmun/cod_ccaa.htm)

- **codiPais** : Codi del país de residència de l'interessat

| **Tipus** | String (3 caràcters) |
| --- | --- |
| **valors d'exemple** | 724 |

Aquest camp contindrà el codi del país de residència de l'interessat en format _ISO 3166-1_ numèric.

Aquests codis es poden consultar a la següent adreça web:

[https://es.wikipedia.org/wiki/ISO\_3166-1](https://es.wikipedia.org/wiki/ISO_3166-1)

- **pais** : Nom del país de residència de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Austràlia |

Aquest camp contindrà el nom del país de residència de l'interessat.

- **adressaCompleta** : Adreça postal completa de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Carrer de Girona 335, 4rt 3a, Barcelona |

Aquest camp contindrà l'adreça postal completa l'interessat.

- **tipusVia** : Tipus de via per a l'adreça postal de l'interessat

| **Tipus** | Enumeració |
| --- | --- |
| **valors d'exemple** | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 40, 41, 42, 99 |

Aquest camp contindrà el tipus de via dins de l'adreça postal de l'interessat. Els possibles valors d'aquest camp son:

| - | - | - | - |
| --- | --- | --- | --- |
| **1** | Alameda | **2** | Calle |
| **3** | Camino | **4** | Carrer |
| **5** | Carretera | **6** | Glorieta |
| **7** | Kalea | **8** | Pasaje |
| **9** | Paseo | **10** | Plaça |
| **11** | Plaza | **12** | Rambla |
| **13** | Ronda | **14** | Rúa |
| **15** | Sector | **16** | Travesía |
| **17** | Urbanización | **18** | Avenida |
| **19** | Avinguda | **20** | Barrio |
| **21** | Calleja | **22** | Camí |
| **23** | Campo | **24** | Carrera |
| **25** | Cuesta | **26** | Edificio |
| **27** | Enparantza | **28** | Estrada |
| **29** | Jardines | **30** | Jardins |
| **31** | Parque | **32** | Passeig |
| **33** | Praza | **34** | Plazuela |
| **35** | Placeta | **36** | Poblado |
| **37** | Via | **38** | Travessera |
| **40** | Passatge | **41** | Bulevar |
| **42** | Polígono | **99** | Otros |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **nomVia** : Nom de la via de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Balmes |

Aquest camp contindrà el nom del carrer de l'adreça postal de l'interessat.

- **numeroSuperior** : Número superior de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | 336Bis |

Aquest camp contindrà el número superior de l'adreça postal de l'interessat.

- **numeroInferior** : Número inferior de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | 334 |

Aquest camp contindrà el número inferior de l'adreça postal de l'interessat.

- **bloc** : Identificador del bloc de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | B |

Aquest camp contindrà l'identificador del bloc de l'adreça postal de l'interessat.

- **portal** : Identificador del portal de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | 2C |

Aquest camp contindrà l'identificador del portal de l'adreça postal de l'interessat.

- **escala** : Identificador de la escala de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | C |

Aquest camp contindrà l'identificador de la porta de l'adreça postal de l'interessat.

- **pis** : Identificador del pis de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | 4rt |

Aquest camp contindrà l'identificador del pis de l'adreça postal de l'interessat.

- **porta** : Identificador de la porta de l'adreça postal de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | 4rt |

Aquest camp contindrà l'identificador de la porta de l'adreça postal de l'interessat.

- **codiPostal** : Codi postal de l'adreça de l'interessat

| **Tipus** | String (5 caràcters) |
| --- | --- |
| **valors d'exemple** | 08008 |

Aquest camp contindrà el codi postal de l'adreça de l'interessat.

- **email** : Adreça de correu electrònic de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | adreca.correu@domini.com |

Aquest camp contindrà l'adreça de correu electrònic de l'interessat.

- **telefonFix** : Número de telèfon fix de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | +34 555 55 66 77 |

Aquest camp contindrà el número de telèfon fix de l'interessat.

- **telefonMobil** : Número de telèfon mòbil de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | +34 555 55 66 77 |

Aquest camp contindrà el número de telèfon mòbil de l'interessat.

- **fax** : Número de fax mòbil de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | +34 555 55 66 77 |

Aquest camp contindrà el número de fax de l'interessat.

- **adressaElectronicaHabilitada** : Adreça electrònic habilitada (DEH) de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** |

Aquest camp contindrà l'adreça electrònica habilitada (DEH) de l'interessat.

- **canalPreferentComunicacio** : Canal preferent de comunicació amb l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | email, sms, aeh, postal |

Aquest camp s'usarà per a informar el canal preferent de comunicació amb l'interessat. Els valors possibles per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **email** | Correu electrònic |
|| _La comunicació amb l'interessat es realitzarà mitjançant un missatge de correu electrònic a l'adreça de l'interessat (camp 'email')._ |
| **sms** | SMS |
|| _La comunicació amb l'interessat es realitzarà mitjançant un missatge SMS al telèfon mòbil de l'interessat (camp 'telefonMobil')._ |
| **aeh** | Adreça electrònica habilitada |
|| _La comunicació amb l'interessat es realitzarà mitjançant l'ús de la seva adreça electrònica habilitada (camp 'adrecaElectronicaHabilitada')._ |
| **postal** | Adreça postal |
|| _La comunicació amb l'interessat es realitzarà mitjançant l'ús de la seva adreça postal (camps 'adressaCompleta' o bé 'tipusVia' + 'nomVia' + 'numeroSuperior' + 'numeroInferior' + 'bloc' + 'portal' + 'escala' + 'pis' + 'porta')._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **consentimentConsulta** : Indicador de consentiment de consulta

| **Tipus** | Boolean |
| --- | --- |
| **valors d'exemple** | true, false |

Aquest camp indica si l'interessat dona el seu consentiment exprés per a que es pugui fer el tractament de les seves dades personals per a la tramitació del procediment.

- **consentimentLOPD** : Indicador de consentiment LOPD

| **Tipus** | Boolean |
| --- | --- |
| **valors d'exemple** | true, false |

Aquest camp indica si la LOPD obliga a que l'interessat hagi de donar el seu consentiment exprés per a poder realitzar el tractament de les seves dades personals.

- **nacionalitat** : Nacionalitat de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Anglesa, Francesa |

Aquest camp s'usarà per a informar-hi la nacionalitat de l'interessat.

- **edat** : Edat de l'interessat

| **Tipus** | Integer |
| --- | --- |
| **valors d'exemple** | 28, 54 |

Aquest camp s'usarà per a informar-hi l'edat de l'interessat

- **dataNaixement** : Data de naixement de l'interessat

| **Tipus** | Long |
| --- | --- |
| **valors d'exemple** | 691671300000 |

Aquest camp s'usarà per a informar-hi la data de naixement de l'interessat, expressada en mil·lisegons des de l'1 de gener de 1970 (epoch).

Al valor d'exemple, tindríem representada amb el número 691671300000 la data corresponent al dilluns 02 de desembre de 1991 a les 11:55:00 GMT+0100.

- **observacions** : Comentaris addicionals

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | L'interessat demana que se'l truqui per telèfon per a informar-lo de... |

Aquest camp s'usarà per a informar-hi un breu comentari amb observacions relatives a l'interessat.

- **representant** : Informació relativa al representant de l'interessat

| **Tipus** | Bloc '_Interessat_' |
| --- | --- |
| | |

Aquest camp conté un bloc amb les mateixes dades que l'interessat.

![5](captures/5.png)

        
###### 3.2.2.5.1 Bloc InformacioIdentificador

Aquest bloc **és generat pel propi MUXv3** a partir de l'anàlisi que fa dels camps '_tipusIdentificador_' i '_identificador_' que trobem dins de les dades de l'interessat.

- **valid** : Indica si l'identificador és vàlid

| **Tipus** | Boolean |
| --- | --- |
| **valors** | true |

Aquest camp indica que, arran de l'anàlisi que fa MUXv3, es considera que l'identificador és vàlid i que compleix amb el format esperat.

- **tipusIdentificador** : Aquest camp indica el tipus d'identificador que ha estat detectat per l'anàlisi fet per MUXv3. Normalment aquest camp coincidirà amb el camp '_tipusIdentificador'_ especificat pel integrador al bloc '_Interessat_'.

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | DNI, NIF, NIE, VAT |

Els valors possibles per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **DNI** | Document Nacional d'Identitat |
|| _El document d'identitat proporcionat pel integrador coincideix amb el format del DNI, és a dir, 8 dígits i sense caràcter de control._ |
| **NIF** | Numero d'Identificació Fiscal |
|| _El document d'identitat proporcionat pel integrador coincideix amb el format del NIF. Els codis de tipus CIF queden englobats dins del NIF._ |
| **NIE** | Numero d'Identificació de l'Estranger |
|| _El document d'identitat proporcionat pel integrador coincideix amb el format del NIE._ |
| **VAT** | Número VAT |
|| _El document d'identitat proporcionat pel integrador coincideix amb el format del VAT o número d'identificació fiscal europeu._ |

- **tipusPersona** : Aquest camp indica el tipus de persona que ha estat detectat per l'anàlisi fet per MUXv3.

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | F, J |

Els valors possibles per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **F** | Persona Física |
|| _Les dades d'identitat proporcionades pel integrador, número de document i tipus de document, corresponen a una persona física._ |
| **J** | Persona Jurídica |
|| _Les dades d'identitat proporcionades pel integrador, número de document i tipus de document, corresponen a una persona jurídica._ |

- **tipusEntitat** : Tipus d'entitat de l'interessat

| **Tipus** | String |
| --- | --- |
| **Valor d'exemple** | Persona Física |

Aquest camp conté una descripció del tipus d'entitat de l'interessat que ha detectat l'anàlisi fet pel MUXv3. Els valors que es retornen actualment son:

- Identificador fiscal europeu [${país}]
- Persona física
- Persona física menor de 14 anys
- Persona física de nacionalitat espanyola no resident
- Persona física estrangera sense NIE
- Persona física de nacionalitat estrangera
- En el cas de NIF d'organitzacions o empreses:
  - Societats anònimes
  - Societats de responsabilitat limitada
  - Societats col·lectives
  - Societats comanditàries
  - Comunitats de bens
  - Societats cooperatives
  - Associacions i fundacions
  - Comunitats de propietaris en règim de propietat horitzontal
  - Societats civils
  - Entitats no residents
  - Corporacions locals
  - Organismes públics
  - Congregacions i institucions religioses
  - Òrgans de l'Administració de l'Estat i comunitats autònomes
  - Unions Temporals d'Empreses
  - Societat Agrària de Transformació
  - Establiments permanents d'entitats no residents a Espanya

- **identificador** : Versió normalitzada de l'Identificador de l'interessat

| **Tipus** | String |
| --- | --- |
| **valors** | 11111111H |

Aquest camp conté el valor normalitzat del número de document d'identificació de l'interessat.

Aquesta normalització consisteix en:

- Pas a majúscules de totes les lletres
- Eliminació de tots els caràcters que no siguin [0-9][A-Z]

A continuació es mostra alguns exemples d'aquestes normalitzacions:

| - | - | - |
| --- | --- | --- |
| 11111111-H | ⇒ | 11111111H |
| 11111111-h | ⇒ | 11111111H |
| q-0811075-a | ⇒ | Q0811075A |
| GB 123456789 | ⇒ | GB123456789 |

- **provincia** : Província associada a l'identificador

| **Tipus** | String |
| --- | --- |
| **Valor d'exemple** | Barcelona |

Si el número de document d'identitat és de tipus CIF (NIF d'una organització), MUXv3 informarà a aquest camp el nom de la província associada.

- **observacions** : Motiu pel qual l'anàlisi no ha pogut determinar el tipus d'identificador

| **Tipus** | String |
| --- | --- |
| **Valor d'exemple** | L'identificador sembla ser un VAT Number però no s'ha pogut resoldre el país al qual pertany |

Quan l'anàlisi no pot determinar el tipus d'identificador, aquest camp contindrà un missatge descriptiu especificant de quin tipus sembla ser aquest i el motiu pel qual no s'ha pogut determinar.

A continuació es mostra un exemple de bloc _Interessats_

```json
"interessats":[
   {
      "tipusIdentificador":{
         "clau":"NIF",
         "descripcio":"Número d'Identificació Fiscal"
      },
      "identificador":"11111111H",
      "informacioIdentificador":{
         "valid":true,
         "tipusIdentificador":"NIF",
         "tipusPersona":"F",
         "tipusEntitat":"Persona física",
         "identificador":"11111111H"
      },
      "dir3personaJuridica":"ES11111111H0000",
      "raoSocial":"Raó Social S.A.",
      "nomSentit":"Pep",
      "nom":"Josep",
      "cognom1":"Cognom1",
      "cognom2":"Cognom2",
      "genere":{
         "clau":"NB",
         "descripcio":"No binari"
      },
      "sexe":"H",
      "codiMunicipi":"080193",
      "municipi":"Barcelona",
      "codiProvincia":"08",
      "provincia":"Barcelona",
      "codiComunitatAutonoma":"09",
      "comunitatAutonoma":"Catalunya",
      "codiPais":"724",
      "pais":"Espanya",
      "adressaCompleta":"Adreça completa 33, Bloc B, Esc C, 1er 3a",
      "tipusVia":{
         "clau":"4",
         "descripcio":"Carrer"
      },
      "nomVia":"Completa",
      "numeroSuperior":"33",
      "numeroInferior":"33",
      "bloc":"B",
      "portal":"3",
      "escala":"K",
      "pis":"1",
      "porta":"3",
      "codiPostal":"08008",
      "email":"adressa@email.com",
      "telefonFix":"555-345-345",
      "telefonMobil":"555-345-345",
      "fax":"555-345-345",
      "adressaElectronicaHabilitada":"adressa@deh.com",
      "canalPreferentComunicacio":"email",
      "consentimentConsulta":true,
      "consentimentLOPD":false,
      "nacionalitat":"Nacionalitat",
      "edat":30,
      "dataNaixement":"669136178000",
      "observacions":"Observacions sobre l'interessat",
      "representant":{
         "tipusIdentificador":{
            "clau":"VAT",
            "descripcio":"Identificador Fiscal Europeu"
         },
         "identificador":"FR0234764537223",
         "informacioIdentificador":{
            "valid":true,
            "tipusIdentificador":"VAT",
            "tipusPersona":"J",
            "tipusEntitat":"Identificador fiscal europeu [França]",
            "identificador":"FR0234764537223"
         },
         "dir3personaJuridica":"L11111111H",
         "raoSocial":"Raó Social Representant S.A.",
         "nomSentit":"Quico",
         "nom":"Francesc",
         "cognom1":"Cognom1",
         "cognom2":"Cognom2",
         "genere":{
            "clau":"NB",
            "descripcio":"No binari"
         },
         "sexe":"H",
         "codiMunicipi":"080193",
         "municipi":"Barcelona",
         "codiProvincia":"08",
         "provincia":"Barcelona",
         "codiComunitatAutonoma":"09",
         "comunitatAutonoma":"Catalunya",
         "codiPais":"724",
         "pais":"Espanya",
         "adressaCompleta":"Adreça completa 33, Bloc B, Esc C, 1er 3a",
         "tipusVia":{
            "clau":"4",
            "descripcio":"Carrer"
         },
         "nomVia":"Completa",
         "numeroSuperior":"33",
         "numeroInferior":"33",
         "bloc":"B",
         "portal":"3",
         "escala":"K",
         "pis":"1",
         "porta":"3",
         "codiPostal":"08008",
         "email":"adressa@email.com",
         "telefonFix":"555 345 345",
         "telefonMobil":"555 345 345",
         "fax":"555 345 345",
         "adressaElectronicaHabilitada":"adressa@deh.com",
         "canalPreferentComunicacio":"email",
         "consentimentConsulta":true,
         "consentimentLOPD":false,
         "nacionalitat":"Nacionalitat",
         "edat":30,
         "dataNaixement":"669136178000",
         "observacions":"Observacions sobre el representant"
      }
   }
]
 ```

    
### 3.2.3 Bloc _Documents_ <a name="3.2.3"></a>

      
##### 3.2.3.1 Documents i fitxers

MUXv3 es recolza en el servei DESA'L com a repositori documental. Això permet que MUXv3 es beneficiï de l'ús d'un servei d'emmagatzemament escalable, virtualment il·limitat i segur. És per això que a MUXv3 també es fa la distinció entre _documents_ i _fitxers._

Un document és una unitat d'informació lògica que conté una sèrie de dades, que en el món DESA'L anomenem _metadades_, i que pot estar relacionada o no amb un _fitxer_, que constitueix el contingut del document en sí. Exemples de fitxer son una fotografia en format JPEG, una presentació en _PowerPoint_, una fulla de càlcul _Excel_, un fitxer _PDF_, etc... El document seria un petit conjunt de dades que proporciona informació addicional i context al fitxer dintre del procediment administratiu.

![6](captures/6.png)

A partir del UUID del document, MUXv3 recupera la informació del document de DESA'L i la incorpora a la transacció.

Hi ha tres tipus de documents, segons el contingut del fitxer estigui custodiat a DESA'L o no:

- Document Intern: _El contingut del fitxer es troba a DESA'L i es pot descarregar mitjançant la API destinada a tal efecte._
- Document Extern per URL: _DESA'L no disposa del contingut del fitxer i aquest es referencia mitjançant una URL. Per a descarregar el contingut del fitxer serà necessari invocar aquesta URL i complir amb els requeriments de seguretat que estableixi._
- Document Extern per CSV: _DESA'L no disposa del contingut del fitxer i aquest es referencia mitjançant un Codi Segur de Verificació. Per a descarregar el contingut del fitxer serà necessari accedir al servei de custodia al qual pertany el CSV i complir amb els requeriments establerts._


##### 3.2.3.2 Documents

Aquest bloc està format per un _map_ amb els documents adjunts a la transacció de MUXv3. Els valors clau d'aquest map son els UUID que identifiquen cadascun dels documents.

![7](captures/7.png)


###### 3.2.3.2.1 Document

Aquest bloc conté tota la informació referent a un document adjunt a la transacció de MUXv3.

A continuació es detallen els camps que conformen el document:

- **uuid** : Identificador únic universal del document

| **Tipus** | String (UUID) |
| --- | --- |
| **valor d'exemple** | ba51a9be-ad53-4fc4-ba90-53355456e2cf |

Aquest camp identifica de manera unívoca el document dins de DESA'L. Per a realitzar qualsevol operació posterior sobre el document, com la descàrrega del seu contingut o la modificació de les seves metadades a DESA'L, caldrà usar aquest identificador.

- **CSVSignatura** : Codi Segur de Verificació de la signatura

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 96435c552ebca40cb0 |

Aquest camp s'usarà per a informar el codi CSV del fitxer que conté la signatura _detached_ del contingut d'aquest document.

- **CSV** : Codi Segur de Verificació del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 96435c552ebca40cb0 |

Aquest camp s'usarà per a informar el codi CSV del propi document, ja estigui custodiat a DESA'L o a un sistema extern.

- **dataAlta** : Data d'alta del document

| **Tipus** | Data |
| --- | --- |
| **valor d'exemple** | 15/04/2021 00:00:00 |

Aquest camp s'usarà per a informar la data d'alta del document a DESA'L.

- **dataDocument** : Data de creació del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 15/04/2021 00:00:00 |

Aquest camp s'usarà per a informar la data de creació del document.

- **descripcio** : Breu descripció de l'objecte del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Formulari de tramitació de la llicència de pesca recreativa |

Aquest camp s'usarà per a informar-hi una breu descripció de l'objecte o finalitat del document.

- **estatElaboracio** : Estat d'elaboració del document

| **Tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | EE01, EE02, EE99, EE03, EE04, EE05 |

Aquest camp indicarà l'estat d'elaboració del document. Els possibles valors per a aquest camp es mostren a continuació:

| - | - |
| --- | --- |
| **EE01** | Original |
|| _El document adjuntat és l'original._ |
| **EE02** | Copia electrònica autèntica amb canvi format |
|| _El document adjuntat és una còpia autèntica de l'original amb una modificació de format respecte a l'original._ |
| **EE03** | Còpia electrònica autèntica document paper |
|| _El document adjuntat és la còpia autèntica electrònica d'un document original en paper._ |
| **EE04** | Còpia electrònica autèntica parcial |
|| _El document adjuntat conté una còpia autèntica formada per fragments del document original._ |
| **EE05** | Altres |
|| _Es coneix l'estat d'elaboració del document adjuntat però no és cap dels anteriors._ |
| **EE99** | Desconegut |
|| _Es desconeix l'estat d'elaboració del document adjuntat._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **formatFitxer** : Tipus MIME del fitxer de document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | application/pdf |

Aquest camp recollirà el tipus MIME del fitxer associat al document adjunt.

- **hash** : Resum criptogràfic del contingut del fitxer de document

| **Tipus** | Base64 |
| --- | --- |
| **valor d'exemple** | UnZbu+PW4J1MBrGlf/DuqgI8S11JT/G0Gw== |

Aquest camp recollirà el resum criptogràfic del contingut del fitxer de document, preferentment de tipus SHA-256.

- **hashAlgoritme** : Algorisme de càlcul del resum criptogràfic

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | SHA256, SHA1 |

Aquest camp recollirà el nom de l'algoritme aplicat per a calcular el resum criptogràfic del contingut del fitxer associat al document.

- **identificadorDocumentOrigen** : Identificador del document en origen

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | DOC0000001 |

Aquest camp recollirà l'identificador del document donat en origen per la entitat que l'ha creat.

- **identificadorDocumentExtern** : Identificador del document extern

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | EXTDOC000002 |

Aquest camp recollirà l'identificador del document extern (és a dir, no custodiat pel DESA'L).

- **nomFitxer** : Nom del fitxer

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | sol\_llicencia\_pesca\_X01234567.pdf |

Aquest camp recollirà el nom del fitxer associat al document.

- **nomNatural** : Nom natural del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Sol·licitud de llicència de pesca recreativa |

Aquest camp recollirà el nom natural donat al document.

- **origen** : Entitat de la qual prové el document

| **Tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | 0, 1 |

Aquest camp indica quina entitat ha originat el document. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **0** | Administració |
|| _El document ha estat originat per la administració pública._ |
| **1** | Ciutadà |
|| _El document ha estat originat pel ciutadà._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **referenciaSignatura** : Identificador del document de signatura

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 93cbf258-593d-4c84-bf12-1abd501bce91 |

Aquest camp indica l'identificador del document que conté la signatura _detached_ per al contingut d'aquest document. Aquesta és una referència _feble_ que DESA'L no verifica pel que es pot donar el cas que el document referenciat no existeixi (per exemple perquè hagi estat esborrat).

- **regulacioGeneracioCSVSignatura** : Normativa aplicada a la generació del CSV

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | El valor del CSV ha estat calculat seguint la normativa aplicable... |

Aquest camp conté una breu referència o cita de la normativa o regulació seguida per a generar el Codi Segur de Verificació del document.

- **tamany** : Mida del fitxer en bytes

| **Tipus** | Long |
| --- | --- |
| **valor d'exemple** | 104933423 |

Aquest camp conté la grandària, en bytes, del fitxer del contingut associat al document.

- **tipusDocumental** : Tipus de document segons estableix la NTI

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | TD99.25, TD99.7, TD99.10, TD99.13, TD99.38, TD99.5, TD99.11, TD99.40, TD23, TD07, TD99.8, TD99.19, TD99.27, TD99.26, TD99.12, TD03, ... |

Aquest camp indica de quin tipus de document es tracta, segons la classificació establerta per la Norma Tècnica d'Interoperabilitat (NTI). Els valors actualment acceptats es mostren a continuació

| **Codi** | **Descripció** |
| --- | --- |
| TD99.25 | Fitxa |
| TD99.7 | Ofici |
| TD99.10 | Avaluació |
| TD99.13 | Burofax |
| TD99.38 | Rebut |
| TD99.5 | Carta |
| TD99.11 | Aportació documentació |
| TD99.40 | Reglament |
| TD23 | Sol·licituds i comunicacions al personal |
| TD07 | Notificació |
| TD99.8 | Citació |
| TD99.19 | Document administratiu |
| TD99.27 | Instrucció |
| TD99.26 | Fotografia |
| TD99.12 | Plica |
| TD03 | Contracte |
| TD99.23 | Extracte |
| TD52 | Moció |
| TD99.33 | Pla |
| TD00 | Altres |
| TD99.43 | Saluda |
| TD10 | Acta |
| TD16 | Al·legació |
| TD99.31 | Nomenament |
| TD61 | Resposta |
| TD04 | Conveni |
| TD64 | Proposta de resolució |
| TD99.2 | Reclamació |
| TD13 | Informe |
| TD14 | Sol·licitud |
| TD99.9 | Document certificat |
| TD99.15 | Acreditació |
| TD17 | Recurs |
| TD99.29 | Invitació |
| TD99.37 | Queixa |
| TD99.14 | Iniciativa parlamentària |
| TD99.28 | Inventari |
| TD99.30 | Llista / llistat |
| TD99.24 | Fax |
| TD99.21 | Estadística |
| TD54 | Convocatòria |
| TD01 | Resolució |
| TD99.41 | Requeriment |
| TD99.36 | Projecte |
| TD19 | Factura |
| TD99.35 | Programa |
| TD18 | Comunicació al ciutadà |
| TD12 | Diligència |
| TD99.42 | Retall de premsa |
| TD11 | Certificat |
| TD99.39 | Registre |
| TD99 | Desconegut |
| TD99.20 | Enquesta |
| TD99.22 | Estudi |
| TD06 | Comunicació |
| TD99.34 | Pressupost |
| TD99.44 | Telegrama |
| TD02 | Acord |
| TD08 | Publicació |
| TD15 | Denuncia |
| TD99.17 | Catàleg |
| TD99.16 | Albarà |
| TD99.18 | Circular |
| TD09 | Acusament de rebuda |
| TD99.32 | Pacte |
| TD20 | Altres incautats |
| TD05 | Declaració |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **tipusDocumentalSICRES** : Tipus de document segons la norma SICRES

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | 01, 02, 03 |

Aquest camp recull el tipus del document segons la especificació SICRES. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **01** | Formulari |
|| _El document correspon al formulari de sol·licitud._ |
| **02** | Document adjunt al formulari |
|| _El document correspon a un adjunt al formulari de sol·licitud presentat._ |
| **03** | Fitxer tècnic intern |
|| _El document és de tipus tècnic i per a un us intern._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **tipusSignatura** : Tipus de signatura aplicat al document

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | TF03, TF02, TF01, TF04, TF06, TF05 |

Aquest camp recull el tipus de signatura que ha estat aplicada al document segons es recull a la NTI. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **TF01** | CSV |
|| _El document ha estat signat mitjançant un CSV._ |
| **TF02** | XAdES internally detached signature |
|| _El document ha estat signat amb una signatura XML en un fitxer apart._ |
| **TF03** | XAdES enveloped signature |
|| _El document ha estat signat amb una signatura XML dins del propi document._ |
| **TF04** | CAdES detached/explicit signature |
|| _El document ha estat signat amb una signatura CMS en un fitxer apart._ |
| **TF05** | CAdES attached/implicit signature |
|| _El document ha estat signat amb una signatura CMS dins del propi document._ |
| **TF06** | PAdES |
|| _El document PDF ha estat signat amb signatura avançada PAdES._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **URLDocumentExtern** : URL per a accedir al fitxer extern

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | https://www.aoc.cat/serveis-aoc/valid/ |

Aquest camp conté, en el cas de documents externs, la URL mitjançant la qual es pot descarregar el contingut del document (fitxer). Val a dir que aquesta URL no és controlada per DESA'L o MUXv3 pel que la seva validesa, requeriments d'autenticació o gestió de les comunicacions no son garantits.

- **versioNTI** : Versió de la NTI que observa el document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | [http://administracionelectronica.gob.es/ENI/XSD/v1.0/documento-e](http://administracionelectronica.gob.es/ENI/XSD/v1.0/documento-e) |

Aquest camp conté l'identificador de la versió de la NTI que observa el document. L'identificador de versió te la forma de URL.

- **identificador** : Identificador del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | ES08134711A\_20210420\_13AC3ABD34 |

Aquest camp conté l'identificador formal donat al document.

- **organ** : Òrgan emissor del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 982192002 |

Aquest camp recull l'identificador de l'organisme que ha creat el document.

- **nivellAcces** : Nivell de seguretat per a l'accés al document

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | A, B, C, E |

Aquest camp recull el nivell de seguretat en l'accés que té el document. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **A** | Secret |
|| _El document té la consideració de document secret._ |
| **B** | Reservat |
|| _El document té la consideració de document reservat._ |
| **C** | Confidencial |
|| _El document té la consideració de document confidencial._ |
| **E** | No classificat |
|| _El document no ha estat classificat._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **classificacioENS** : Classificació del document segons l'ENS

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | Alt, Mig, Baix |

Aquest camp indica la classificació del document segons l'Esquema Nacional de Seguretat (ENS). Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **Alt** | Alt |
|| _El document té la consideració de document de nivell de seguretat alt._ |
| **Mig** | Mig |
|| _El document té la consideració de document de nivell de seguretat mig._ |
| **Baix** | Baix |
|| _El document té la consideració de document de nivell de seguretat baix._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **sensibilitatDadesCaracterPersonal** : Aquest camp recull

| **Tipus** | Enumeració |
| --- | --- |
| **valors** | Alt, Mig, Basic |

Aquest camp indica el nivell de sensibilitat de les dades de caràcter personal que conté el document. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **Alt** | Alt |
|| _El document conté dades personals altament sensibles._ |
| **Mig** | Mig |
|| _El document conté dades personals sensibles._ |
| **Basic** | Bàsic |
|| _El document no conté dades personals sensibles._ |

**\*Aquests valors es troben a una taula mestra i per tant poden canviar en el futur.**

- **documentEssencial** : Indicador de document essencial

| **Tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | true |

Aquest camp indica si el document té la consideració de document essencial o no.

- **idioma** : Idioma del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | en\_US |

Aquest camp recull l'idioma del contingut del document.

- **codiClassificacio** : Codi de classificació del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 00023 |

Aquest camp recull el codi de classificació donat al document.

- **nomClassificacio** : Descripció del codi de classificació

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Classificació |

Aquest camp mostra la breu descripció del codi de classificació informat per al document.

- **codiSIA** : Codi SIA del document

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | SIA001 |

Aquest camp conté el codi SIA (Sistema de Información Administrativa) donat al document. Per a més informació sobre SIA, podeu consultar el següent enllaç:

[https://administracionelectronica.gob.es/ctt/sia/descargas#.YH6As6\_7SUl](https://administracionelectronica.gob.es/ctt/sia/descargas#.YH6As6_7SUl)

- **identificadorExpedientDesal** : Identificador de l'expedient DESA'L

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | ba51a9be-ad53-4fc4-ba90-53355456e2cf |

Aquest camp conté l'identificador de l'expedient, dins de DESA'L, al qual pertany el document. Al igual que per als documents, els identificadors d'expedient a DESA'L tenen el format UUID versió 4.

- **codiINE** : Identificador de l'organisme propietari

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 9821920002 |

Aquest camp conté l'identificador de l'organisme que ha creat (propietari) el document a DESA'L.

- **codiServei** : Identificador del servei propietari

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | eNOTUM, eVALISA |

Aquest camp conté l'identificador del servei que ha creat (propietari) el document a DESA'L.

- **contingut** : Tipus de contingut o document

| **Tipus** | Enumeració |
| --- | --- |
| **valor d'exemple** | 1, 2, 3 |

Aquest camp indica el tipus de contingut del document. Els valors possibles per a aquest camp es detallen a continuació:

| - | - |
| --- | --- |
| **1** | Document intern |
|| _El contingut del document és custodiat a DESA'L._ |
| **2** | Document extern per URL |
|| _El contingut del document no es troba a DESA'L i s'hi ha d'accedir mitjançant una URL._ |
| **3** | Document extern per CSV |
|| _El contingut del document no es troba a DESA'L i s'hi ha d'accedir mitjançant un servei extern de custodia._ |

- **interessat** : Llista amb els identificadors dels interessats

| **Tipus** | List\<String\> |
| --- | --- |
| **valor d'exemple** | [11111111H, 22222222H] |

Aquest camp conté els identificadors dels interessats associats al document, per exemple, els ciutadans que ha emplenat el formulari de sol·licitud.

- **usuari** : Identificador de l'usuari que ha creat el document a DESA'L

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 12333444X |

Aquest camp conté l'identificador de l'usuari que ha creat el document a DESA'L.

- **identificadorExpedientExtern** : Identificador de l'expedient extern a DESA'L

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | SH-2021-000331 |

Aquest camp conté l'identificador d'un expedient , extern a DESA'L al qual pertany el document.

- **numeroRegistre** : Número d'assentament

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | E/2021-0000145 |

Aquest camp conté un número d'assentament associat al document. Aquest camp normalment no tindrà mai un valor en aquest punt, ja que l'assentament es realitzarà precisament amb MUXv3.

- **infoAddicional** : Dades addicionals

| **Tipus** | Map |
| --- | --- |
| **valor d'exemple** | [<br>{"temperatura": "29"},<br>{"longitud": "12.123456"},<br>{"latitud": "10.123456"},<br>{"alcada": "300"}<br>] |

Aquest camp consisteix en un mapa clau/valor que permet incorporar al document dades que l'usuari de DESA'L ha cregut necessari afegir.

A continuació es mostra un exemple de bloc _Documents_

```json
 "documents": {
  "ba51a9be-ad53-4fc4-ba90-53355456e2cf": {
    "dataAlta": 1601905334468,
    "dataDocument": 1601905334468,
    "descripcio": "Document extern de prova",
    "estatElaboracio": {
      "clau": "EE99",
      "descripcio": "Desconegut"
    },
    "formatFitxer": "application/pdf",
    "hash": "YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXowMTIzMDE=",
    "identificadorDocumentOrigen": "ID000012021GENCAT33BA2F",
    "nomFitxer": "solicitud_permís_crema_controlada.pdf",
    "nomNatural": "Sol·licitud de permís per a la cura i manteniment de masses forestals",
    "origen": {
      "clau": "1",
      "descripcio": "Ciutadà"
    },
    "referenciaSignatura": "11111111-3241-f3f2-fec3-3451d1eaf3a1",
    "regulacioGeneracioCSVSignatura": "Disposició 322 llei 23/2021",
    "tamany": 400000,
    "tipusDocumental": {
      "clau": "TD99.11",
      "descripcio": "Aportació documentació"
    },
    "tipusDocumentalSICRES": {
      "clau": "02",
      "descripcio": "Document adjunt al formulari"
    },
    "tipusSignatura": {
      "clau": "TF01",
      "descripcio": "CSV"
    },
    "uuid": "ba51a9be-ad53-4fc4-ba90-53355456e2cf",
    "versioNTI": "http://administracionelectronica.gob.es/ENI/XSD/v1.0/documento-e",
    "identificador": "ES_9821920002_A00033171_000000001",
    "organ": "9821920002",
    "nivellAcces": {
      "clau": "E",
      "descripcio": "No classificat"
    },
    "classificacioENS": {
      "clau": "A",
      "descripcio": "Alt"
    },
    "sensibilitatDadesCaracterPersonal": {
      "clau": "A",
      "descripcio": "Alt"
    },
    "documentEssencial": false,
    "idioma": "ca_ES",
    "codiClassificacio": "000324455",
    "nomClassificacio": "TRAMESA GENERICA",
    "codiSIA": "SIA000001",
    "nomSIA": "TRAMESA GENERICA",
    "identificadorExpedientDesal": "c1c1c1c1-b2b2-c3c3-d4d4-e5e5e5e5e5e5",
    "codiINE": "9821920002",
    "codiServei": "eVALISA",
    "contingut": 2,
    "interessat": [
      "11111111H"
    ],
    "usuari": "11222222H",
    "identificadorExpedientExtern": "IDEXP_EXTERN_2021_MARC_000001",
    "CSVSignatura": "SIG-98219200020503202144A",
    "CSV": "98219200020503202133ACD2CA",
    "URLDocumentExtern": "https://www.google.com"
  },
  "93cbf258-593d-4c84-bf12-1abd501bce91": {
    "dataAlta": 1601905334468,
    "dataDocument": 1601905334468,
    "descripcio": "Document de prova #2",
    "estatElaboracio": {
      "clau": "EE99",
      "descripcio": "Desconegut"
    },
    "formatFitxer": "application/pdf",
    "hash": "YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXowMTIzMDE=",
    "identificadorDocumentOrigen": "ID000012021GENCAT33BA2F",
    "nomFitxer": "documentació_adjunta_solicitud.pdf",
    "nomNatural": "Aportació de documentació requerida per a la sol·licitud del permís de cremes controlades ",
    "origen": {
      "clau": "1",
      "descripcio": "Ciutadà"
    },
    "referenciaSignatura": "22222222-3241-f3f2-fec3-3451d1eaf3a1",
    "regulacioGeneracioCSVSignatura": "Disposició 322 llei 23/2021",
    "tamany": 400000,
    "tipusDocumental": {
      "clau": "TD99.11",
      "descripcio": "Aportació documentació"
    },
    "tipusDocumentalSICRES": {
      "clau": "02",
      "descripcio": "Document adjunt al formulari"
    },
    "tipusSignatura": {
      "clau": "TF01",
      "descripcio": "CSV"
    },
    "uuid": "93cbf258-593d-4c84-bf12-1abd501bce91",
    "versioNTI": "http://administracionelectronica.gob.es/ENI/XSD/v1.0/documento-e",
    "identificador": "ES_9821920002_A00033171_000000001",
    "organ": "9821920002",
    "nivellAcces": {
      "clau": "E",
      "descripcio": "No classificat"
    },
    "classificacioENS": {
      "clau": "A",
      "descripcio": "Alt"
    },
    "sensibilitatDadesCaracterPersonal": {
      "clau": "A",
      "descripcio": "Alt"
    },
    "documentEssencial": false,
    "idioma": "ca_ES",
    "codiClassificacio": "000324455",
    "nomClassificacio": "TRAMESA GENERICA",
    "codiSIA": "SIA000001",
    "nomSIA": "TRAMESA GENERICA",
    "identificadorExpedientDesal": "EXP20210300001",
    "codiINE": "9821920002",
    "codiServei": "eVALISA",
    "contingut": 1,
    "interessat": [
      "11111111H"
    ],
    "usuari": "33333333H",
    "identificadorExpedientExtern": "IDEXP_EXTERN_2021_MARC_000001",
    "CSVSignatura": "SIG-98219200020503202144B",
    "CSV": "98219200020503202133AC2C3"
  }
}

 ```

    
### 3.2.4 Bloc _Registres_ <a name="3.2.4"></a>

Aquest bloc conté les dades relatives als organismes als que va dirigit l'assentament_._

Els camps que formen aquest bloc es descriuen a continuació:

- **origen** : Aquest bloc conté les dades de l'organisme que emet l'assentament.

| **Tipus** | Bloc Registre |
| --- | --- |

- **desti** : Aquest bloc conté les dades de l'organisme al que va dirigit l'assentament.

| **Tipus** | Bloc Registre |
| --- | --- |

![8](captures/8.png)

      1.
##### Bloc Registre

Aquest bloc conté les dades que defineixen la unitat orgànica i la oficina de registre des del qual s'origina o a qui va dirigit l'assentament.

Els camps que formen aquest bloc es detallen a continuació:

- **unitatOrganica** : Unitat orgànica

| **Tipus** | Bloc _UnitatOrganica_ |
| --- | --- |

Aquest bloc conté les dades de la unitat orgànica corresponent a l'organisme segons es recullen al sistema DIR3.

- **unitatTramitadora** : Unitat tramitadora

| **Tipus** | Bloc _UnitatTramitadora_ |
| --- | --- |

Aquest bloc conté el codi i nom de la unitat tramitadora implicada a l'assentament.

- **oficinaRegistre** : Oficina de Registre

| **Tipus** | Bloc _OficinaRegistre_ |
| --- | --- |

Aquest bloc conté les dades de la oficina de registre corresponent a l'organisme segons es recullen al sistema DIR3.

- **sarcat** : Codis de centre de Sarcat

| **Tipus** | Bloc Sarcat |
| --- | --- |

Aquest bloc conté els camps dirigits a poder especificar a quina unitat, dins de la estructura de Sarcat, va dirigit l'assentament.

       
##### 3.2.4.1.1 Bloc UnitatOrganica

Aquest bloc conté totes les dades referents a la unitat orgànica. En el cas d'unitats orgàniques registrades al sistema DIR3 MUXv3 completarà el bloc amb la jerarquia d'aquestes, així com les oficines de registre associades.

![9](captures/9.png)

- **dir3** : Identificador DIR3 de la unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | E03092701 |

Aquest camp contindrà l'identificador de tipus DIR3 de l'organisme.

- **dir3Pare** : Identificador DIR3 de la unitat pare

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | A02004970 |

Aquest camp conté l'identificador de la unitat orgànica de la qual aquesta depèn.

- **ine10** : Identificador INE10 de la unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 9821920002 |

Aquest camp conté l'identificador INE10 (EACAT) de la unitat orgànica.

- **nom** : Nom de la unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Consorci Administració Oberta de Catalunya |

Aquest camp conté el nom de la unitat orgànica.

- **cif** : CIF de la unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Q2268242A |

Aquest camp conté el CIF de la unitat orgànica.

- **codiExtern** : Codi auxiliar d'identificació

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 3433788A |

Aquest camp conté un identificador propi emprat per la unitat orgànica i que no és cap dels estàndard.

- **codiTipusVia** : Codi del tipus de via de la adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 2 |

Aquest camp indica el codi del tipus de via de l'adreça postal de la unitat orgànica.

- **tipusVia** : Nom o descripció del tipus de via

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Carrer, Gran Via, Rambla |

Aquest camp recull la descripció del tipus de via de l'adreça postal de la unitat orgànica.

- **adressa** : Nom de la via de l'adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Pedro Sesé |

Aquest camp recull el nom de la via de l'adreça postal de la unitat orgànica.

- **numeroVia** : Número de la via de l'adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 33bis, 400-402 |

Aquest camp indica el número de la via de l'adreça postal de la unitat orgànica.

- **codiMunicipi** : Identificador del municipi

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 221992 |

Aquest camp dona l'identificador del municipi en format INE6 (codi provincia + codi municipi + dígit de control) de l'adreça postal de la unitat orgànica.

- **nomMunicipi** : Nom del municipi

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Sabiñánigo |

Aquest camp conté el nom del municipi de l'adreça postal de la unitat orgànica.

- **codiComarca** : Codi de la comarca

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 40 |

Aquest camp conté el codi de comarca de l'adreça postal de la unitat orgànica.

- **nomComarca** : Nom de la comarca

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Vallès Occidental |

Aquest camp conté el nom de la comarca de l'adreça postal de la unitat orgànica.

- **codiProvincia** : Codi de província

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 08, 17, 25, 43 |

Aquest camp conté el codi de província de la adreça postal de la unitat orgànica.

- **nomProvincia** : Nom de la província

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Barcelona, Girona, Tarragona, Lleida |

Aquest camp conté el nom de la província de la adreça postal de la unitat orgànica.

- **codiPostal** : Codi postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 08018, 08293 |

Aquest camp conté el codi postal de l'adreça de la unitat orgànica.

- **codiComunitat** : Codi de comunitat autònoma

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 09 |

Aquest camp indica el codi de la comunitat autònoma a la qual es troba la unitat orgànica.

- **nomComunitat** : Nom de la comunitat autònoma

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Catalunya |

Aquest camp indica el nom de la comunitat autònoma a la qual es troba la unitat orgànica.

- **codiPais** : Codi de país

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 724 |

Aquest camp conté el codi del país al qual es troba la unitat orgànica.

El codi de país segueix el format ISO 3166-1 numèric. Per a més informació es pot consultar el següent enllaç:

[https://en.wikipedia.org/wiki/ISO\_3166-1](https://en.wikipedia.org/wiki/ISO_3166-1)

- **nomPais** : Nom del país

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Francia, Costa Rica |

Aquest camp conté el nom o descripció del país on es troba la unitat orgànica.

- **codiAmbitTerritorial** : Codi d'àmbit territorial

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 13, 14 |

Aquest camp conté el codi d'àmbit territorial corresponent a la unitat orgànica. Aquesta codificació s'obté del sistema DIR3.

- **nomAmbitTerritorial** : Descripció de l'àmbit territorial

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Local, Provincial |

Aquest camp conté la descripció del codi d'àmbit territorial de la unitat orgànica.

- **codiEntitatGeografica** : Codi d'entitat geogràfica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 01, 11 |

Aquest camp conté el codi d'entitat geogràfica que aplica a la unitat orgànica.

- **nomEntitatGeografica** : Nom de la entitat geogràfica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Municipio, Central-Estatal |

Aquest camp conté la descripció del codi d'entitat geogràfica que aplica a la unitat orgànica.

- **codiNivellAdministracio** : Codi del nivell d'administració

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 2 |

Aquest camp conté el codi del nivell d'administració que aplica a la unitat orgànica.

- **nomNivellAdministracio** : Nom del nivell d'administració

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Administración autonómica |

Aquest camp conté la descripció del nivell d'administració que aplica a la unitat orgànica.

- **codiTipusEntitatPublica** : Codi de tipus d'entitat pública

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | CA, AY |

Aquest camp conté el codi de tipus d'entitat pública de la unitat orgànica.

- **nomTipusEntitatPublica** : Nom del tipus d'entitat pública

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Comunitat Autònoma, Ayuntamiento |

Aquest camp conté la descripció del codi de tipus d'entitat pública de la unitat orgànica.

- **codiTipusUnitatOrganica** : Codi de tipus d'unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | UP |

Aquest camp conté el tipus d'unitat orgànica.

- **nomTipusUnitatOrganica** : Nom del tipus d'unitat orgànica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Unidad Principal |

Aquest camp conte la descripció del tipus d'unitat orgànica.

- **dadesContacte** : Informació de contacte

| **Tipus** | Bloc _dadesContacte_ |
| --- | --- |
| **valor d'exemple** | "dadesContacte": {<br>"telefon": [<br>"976761000"<br>],<br>"email": [<br>"registro@unizar.es"<br>]<br>} |

Dins aquest bloc s'hi podrà trobar els números de telèfon, adreces de correu electrònic, planes web o números de fax mitjançant els quals es pot contactar amb la unitat orgànica.

El bloc està organitzat a mode de mapa de llistes, servint com a clau del mapa el tipus de contacte:

| **CLAU** |
| --- |
| telefon |
| fax |
| email |
| URL |
| centraleta |
| altres |

Així per a cadascun dels mitjans de contacte tindrem una llista amb les opcions possibles:

![10](captures/10.png)

- **oficinesAssociades** : Oficines de registre associades a la unitat orgànica

| **Tipus** | Bloc _oficinesAssociades_ |
| --- | --- |

Aquest camp està format per un conjunt (S_et_) que conté totes les oficines de registre associades a la unitat orgànica segons es recull al sistema DIR3.

- **unitatPare** : Dades de la unitat orgànica

| **Tipus** | Bloc _unitatOrganica_ |
| --- | --- |

Aquest bloc conté les dades de la unitat orgànica responsable (pare) d'aquesta unitat orgànica. La cadena segueix fins a arribar a la unitat orgànica arrel

          
###### 3.2.4.1.1 Bloc _OficinesAssociades_

Aquest bloc conté un conjunt d'oficines de registre associades a la unitat orgànica a la que pertany el bloc. Cadascuna d'aquestes oficines de registre està formada per un bloc _OficinaRegistre_.

![11](captures/11.png)

        
##### 3.2.4.1.2 Bloc UnitatTramitadora

Aquest bloc conté dos camps, corresponents al codi i nom de la unitat tramidora implicada a l'assentament.

- **codi** : Identificador de la unitat tramitadora

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 000000001 |

- **nom** : Nom de la unitat tramitadora

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Nom de la unitat tramitadora |

        
##### 3.2.4.1.3 Bloc OficinaRegistre

Cadascun dels blocs _OficinaRegistre_ conté els següents camps:

- **dir3** : Identificador DIR3 de la oficina de registre

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | O00011309 |

Aquest camp contindrà l'identificador de tipus DIR3 de la oficina de registre.

- **dir3Pare** : Identificador DIR3 de la oficina de registre pare

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | O00011309 |

Aquest camp conté l'identificador de la oficina de registre pare d'aquesta.

- **dir3UnitatResponsable** : Identificador DIR3 de la unitat orgànica responsable

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | A02004970 |

Aquest camp conté l'identificador de la unitat orgànica responsable d'aquesta oficina de registre.

No s'ha de confondre la unitat orgànica responsable amb la unitat orgànica a la qual serveix la oficina de registre. La llista del les unitats orgàniques a les que la oficina dona servei es pot trobar al camp _unitatsServides_.

- **ine10** : Identificador INE10 de la oficina de registre

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 9821920002 |

Aquest camp conté l'identificador INE10 (EACAT) de la oficina de registre.

- **nom** : Nom de la oficina de registre

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Consorci Administració Oberta de Catalunya |

Aquest camp conté el nom de la oficina de registre.

- **codiTipusVia** : Codi del tipus de via de la adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 2 |

Aquest camp indica el codi del tipus de via de l'adreça postal de la oficina de registre.

- **tipusVia** : Nom o descripció del tipus de via

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Carrer, Gran Via, Rambla |

Aquest camp recull la descripció del tipus de via de l'adreça postal de la oficina de registre.

- **adressa** : Nom de la via de l'adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Pedro Sesé |

Aquest camp recull el nom de la via de l'adreça postal de la oficina de registre.

- **numeroVia** : Número de la via de l'adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 33bis, 400-402 |

Aquest camp indica el número de la via de l'adreça postal de la oficina de registre.

- **codiMunicipi** : Identificador del municipi

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 221992 |

Aquest camp dona l'identificador del municipi en format INE6 (codi provincia + codi municipi + dígit de control) de l'adreça postal de la oficina de registre.

- **nomMunicipi** : Nom del municipi

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Sabiñánigo |

Aquest camp conté el nom del municipi de l'adreça postal de la oficina de registre.

- **codiComarca** : Codi de la comarca

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 40 |

Aquest camp conté el codi de comarca de l'adreça postal de la oficina de registre.

- **nomComarca** : Nom de la comarca

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Vallès Occidental |

Aquest camp conté el nom de la comarca de l'adreça postal de la oficina de registre.

- **codiProvincia** : Codi de província

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 08, 17, 25, 43 |

Aquest camp conté el codi de província de la adreça postal de la oficina de registre.

- **nomProvincia** : Nom de la província

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Barcelona, Girona, Tarragona, Lleida |

Aquest camp conté el nom de la província de la adreça postal de la oficina de registre.

- **codiPostal** : Codi postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 08018, 08293 |

Aquest camp conté el codi postal de l'adreça de la oficina de registre.

- **codiComunitat** : Codi de comunitat autònoma

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 09 |

Aquest camp indica el codi de la comunitat autònoma a la qual es troba la oficina de registre.

- **nomComunitat** : Nom de la comunitat autònoma

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Catalunya |

Aquest camp indica el nom de la comunitat autònoma a la qual es troba la oficina de registre.

- **codiPais** : Codi de país

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 724 |

Aquest camp conté el codi del país al qual es troba la oficina de registre.

El codi de país segueix el format ISO 3166-1 numèric. Per a més informació es pot consultar el següent enllaç:

[https://en.wikipedia.org/wiki/ISO\_3166-1](https://en.wikipedia.org/wiki/ISO_3166-1)

- **nomPais** : Nom del país

| **Tipus** | String |
| --- | --- |
| **valors d'exemple** | Francia, Costa Rica |

Aquest camp conté el nom o descripció del país on es troba la oficina de registre.

- **codiEntitatGeografica** : Codi d'entitat geogràfica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 01, 11 |

Aquest camp conté el codi d'entitat geogràfica que aplica a la oficina de registre.

- **nomEntitatGeografica** : Nom de la entitat geogràfica

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Municipio, Central-Estatal |

Aquest camp conté la descripció del codi d'entitat geogràfica que aplica a la oficina de registre.

- **codiNivellAdministracio** : Codi del nivell d'administració

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 2 |

Aquest camp conté el codi del nivell d'administració que aplica a la oficina de registre.

- **nomNivellAdministracio** : Nom del nivell d'administració

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Administración autonómica |

Aquest camp conté la descripció del nivell d'administració que aplica a la oficina de registre.

- **observacions** : Observacions generals

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Oficina principal |

Aquest camp conté observacions generals relatives a la oficina de registre.

- **horariAtencio** : Horari d'atenció presencial

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Dilluns de 09:00 a 15:00, Dimarts de 09:00 a 17:00 |

Aquest camp conté un breu text indicant l'horari d'atenció presencial de la oficina de registre.

- **diesInabils** : Dies de tancament

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 25 de desembre, 1 de gener, 6 de gener |

Aquest camp conté un breu text descrivint la llista de dies inhàbils en els quals no es presta atenció presencial.

- **observacionsAdreca** : Observacions relatives a l'adreça postal

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Al costat de l'edifici Glòries |

Aquest camp conté el codi de tipus d'entitat pública de la unitat orgànica.

- **tipusOficina** : Tipus d'oficina de registre

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 1, 0 |

Aquest camp indica si la oficina de registre és la principal (1) o la auxiliar (0).

- **dadesContacte** : Informació de contacte

| **Tipus** | Bloc _dadesContacte_ |
| --- | --- |
| **valor d'exemple** | "dadesContacte": {<br>"telefon": [<br>"913630801"<br>],<br>"fax": [<br>"915360945"<br>],<br>"email": [<br>"ministerio\_trabajo.madrid@correos.com"<br>]<br>} |

Dins aquest bloc s'hi podrà trobar els números de telèfon, adreces de correu electrònic, planes web o números de fax mitjançant els quals es pot contactar amb la oficina de registre.

El bloc està organitzat a mode de mapa de llistes, servint com a clau del mapa el tipus de contacte:

| **CLAU** |
| --- |
| telefon |
| fax |
| email |
| URL |
| centraleta |
| altres |

Així per a cadascun dels mitjans de contacte tindrem una llista amb les opcions possibles:

![12](captures/12.png)

- **serveisOficina** : Llista de serveis suportats per la oficina de registre

| **Tipus** | Llista de blocs _ServeisOficina_ |
| --- | --- |
| **valor d'exemple** | [<br>{"clau": "1", "descripcio": "Oficina de Registro"},<br>{"clau": "5", "descripcio": "Oficina integrada en SIR"}<br>] |

Aquest camp conté una llista amb els serveis que suporta la oficina de registre. Aquests serveis estan definits al _Directorio Común_ de DIR3 i es detallen a continuació.

| **Codi** | **Descripció** |
| --- | --- |
| 1 | Oficina de Registro |
| 2 | Oficina de Información |
| 3 | Oficina de Tramitación |
| 4 | Registro Virtual no Presencial |
| 5 | Oficina Integrada en SIR |
| 6 | Oficina Integrada en SIR (Envío) |
| 7 | Oficina Integrada en SIR (Recepción) |
| 8 | Oficina Integrada 060 |
| 9 | Oficina de Correos |
| 10 | Oficina de Extranjería |
| 11 | Oficina de Violencia de Género |
| 12 | Oficina Accesible |
| 13 | Cl@ve |
| 14 | REA |
| 15 | Oficina ORD |
| 16 | Ref. Única (Recepción) |

- **unitatsServides** : Llista d'unitats orgàniques servides per la oficina de registre

| **Tipus** | Llista de String/Bloc _UnitatOrganica_ |
| --- | --- |

Aquest camp conté una llista de blocs _UnitatOrganica_ amb la informació de totes les unitats orgàniques a les que la oficina de registre presta servei.

![13](captures/13.png)

De no poder-se trobar tota la informació disponible per a l'identificador d'alguna oficina de registre, llavors s'informarà un _string_ amb aquest identificador. Per tant, cal fer un tractament lax d'aquest bloc, que té un objectiu purament informatiu.

- **oficinaResponsable** : Oficina de registre responsable

| **Tipus** | Bloc _OficinaRegistre_ |
| --- | --- |

Aquest camp conté un bloc de tipus _OficinaRegistre_ amb tota la informació de la oficina de registre responsable d'aquesta.

![14](captures/14.png)

        
##### 3.2.4.1.4 Bloc Sarcat

El bloc de dades Sarcat conté els camps necessaris per a poder referenciar les unitats orgàniques i centres de procedència i destinació dins del registre de la Generalitat de Catalunya, que té una codificació a banda de INE10 o DIR3. Cadascun dels blocs _Sarcat_ conté els següents camps:

- **urPK** : Identificador de la unitat de registre

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 0406 |

Aquest camp conté l'identificador de la unitat de registre a Sarcat.

- **idCentreProcedencia** : Identificador del centre de procedència

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 57754 |

Aquest camp conté l'identificador del centre de procedència dins de Sarcat.

- **idCentreDestinacio** : Identificador del centre de destinació

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 32204 |

Aquest camp conté l'identificador del centre de destinació a Sarcat.

- **idCentreDestinacioInterna** : Identificador del centre de destinació

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 4552 |

Aquest camp conté l'identificador del centre de destinació intern a Sarcat.

- **idCentreDestinacioExterna** : Identificador INE10 de la oficina de registre

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 4357 |

Aquest camp conté l'identificador del centre de destinació extern a Sarcat.

A continuació es mostra un exemple del bloc _Registres_:

```json
"registres":{
   "origen":{
      "unitatOrganica":{
         "dir3":"A09018933",
         "dir3Pare":"A09018799",
         "ine10":"9821920002",
         "nom":"Consorci Administració Oberta de Catalunya",
         "cif":"Q0801175A",
         "codiTipusVia":"2",
         "tipusVia":"Calle",
         "adressa":"Tánger",
         "numeroVia":"98",
         "codiMunicipi":"080193",
         "nomMunicipi":"Barcelona",
         "codiComarca":"13",
         "nomComarca":"Barcelonès",
         "codiProvincia":"08",
         "nomProvincia":"Barcelona",
         "codiPostal":"08018",
         "codiComunitat":"09",
         "nomComunitat":"Cataluña",
         "codiPais":"724",
         "nomPais":"España",
         "codiAmbitTerritorial":"12",
         "nomAmbitTerritorial":"Central-Autonómico",
         "codiEntitatGeografica":"01",
         "nomEntitatGeografica":"Municipio",
         "codiNivellAdministracio":"2",
         "nomNivellAdministracio":"Administración Autonómica",
         "oficinesAssociades":[
            {
               "dir3":"O00011864",
               "dir3UnitatResponsable":"A09018933",
               "ine10":"9821920002",
               "nom":"Consorcio Administración Abierta de Cataluña (CAOC)",
               "codiTipusVia":"2",
               "tipusVia":"Calle",
               "adressa":"Tanger",
               "numeroVia":"98",
               "codiMunicipi":"080193",
               "nomMunicipi":"Barcelona",
               "codiComarca":"13",
               "nomComarca":"Barcelonès",
               "codiProvincia":"08",
               "nomProvincia":"Barcelona",
               "codiPostal":"08018",
               "codiComunitat":"09",
               "nomComunitat":"Cataluña",
               "codiPais":"724",
               "nomPais":"España",
               "codiEntitatGeografica":"01",
               "nomEntitatGeografica":"Municipio",
               "codiNivellAdministracio":"2",
               "nomNivellAdministracio":"Administración Autonómica",
               "serveisOficina":[
                  {
                     "codi":"5",
                     "descripcio":"Oficina Integrada en SIR"
                  },
                  {
                     "codi":"6",
                     "descripcio":"Oficina Integrada en SIR (Envío)"
                  },
                  {
                     "codi":"7",
                     "descripcio":"Oficina Integrada en SIR (Recepción)"
                  }
               ],
               "unitatsServides":[
                  "A09018933"
               ]
            }
         ],
         "unitatPare":{
            "dir3":"A09018799",
            "dir3Pare":"A09002970",
            "ine10":"9610420002",
            "nom":"Departamento de Políticas Digitales y Administraciones Públicas",
            "cif":"S0811001G",
            "codiTipusVia":"37",
            "tipusVia":"Via",
            "adressa":"Laietana",
            "numeroVia":"26",
            "codiMunicipi":"080193",
            "nomMunicipi":"Barcelona",
            "codiComarca":"13",
            "nomComarca":"Barcelonès",
            "codiProvincia":"08",
            "nomProvincia":"Barcelona",
            "codiPostal":"08003",
            "codiComunitat":"09",
            "nomComunitat":"Cataluña",
            "codiPais":"724",
            "nomPais":"España",
            "codiAmbitTerritorial":"12",
            "nomAmbitTerritorial":"Central-Autonómico",
            "codiEntitatGeografica":"01",
            "nomEntitatGeografica":"Municipio",
            "codiNivellAdministracio":"2",
            "nomNivellAdministracio":"Administración Autonómica",
            "codiTipusUnitatOrganica":"CJ",
            "nomTipusUnitatOrganica":"Consejería",
            "oficinesAssociades":[
               {
                  "dir3":"O00007257",
                  "dir3UnitatResponsable":"A09018799",
                  "ine10":"9610420002",
                  "nom":"Departamento de Políticas Digitales y Administraciones Públicas",
                  "codiTipusVia":"37",
                  "tipusVia":"Via",
                  "adressa":"Laietana",
                  "numeroVia":"26",
                  "codiMunicipi":"080193",
                  "nomMunicipi":"Barcelona",
                  "codiComarca":"13",
                  "nomComarca":"Barcelonès",
                  "codiProvincia":"08",
                  "nomProvincia":"Barcelona",
                  "codiPostal":"08003",
                  "codiComunitat":"09",
                  "nomComunitat":"Cataluña",
                  "codiPais":"724",
                  "nomPais":"España",
                  "codiEntitatGeografica":"01",
                  "nomEntitatGeografica":"Municipio",
                  "codiNivellAdministracio":"2",
                  "nomNivellAdministracio":"Administración Autonómica",
                  "tipusOficina":"1",
                  "dadesContacte":{
                     "telefon":[
                        "935671700"
                     ],
                     "fax":[
                        "935671769"
                     ]
                  },
                  "serveisOficina":[
                     {
                        "codi":"1",
                        "descripcio":"Oficina de Registro"
                     },
                     {
                        "codi":"5",
                        "descripcio":"Oficina Integrada en SIR"
                     },
                     {
                        "codi":"7",
                        "descripcio":"Oficina Integrada en SIR (Recepción)"
                     }
                  ],
                  "unitatsServides":[
                     "A09018799"
                  ]
               }
            ],
            "unitatPare":{
               "dir3":"A09002970",
               "dir3Pare":"A99999999",
               "ine10":"0000000001",
               "nom":"Generalitat de Catalunya",
               "cif":"S0811001G",
               "codiMunicipi":"080193",
               "codiComarca":"13",
               "nomComarca":"Barcelonès",
               "codiProvincia":"08",
               "codiPostal":"08003",
               "codiAmbitTerritorial":"12",
               "nomAmbitTerritorial":"Central-Autonómico",
               "codiNivellAdministracio":"2",
               "nomNivellAdministracio":"Administración Autonómica",
               "codiTipusEntitatPublica":"CA",
               "nomTipusEntitatPublica":"Comunidad Autonoma",
               "codiTipusUnitatOrganica":"PR",
               "nomTipusUnitatOrganica":"Presidencia",
               "dadesContacte":{
                  "telefon":[
                     "934024600"
                  ],
                  "fax":[
                     "933183488"
                  ],
                  "URL":[
                     "http://www.president.cat/pres\\_gov/president/ca/president-mas/contactar.html"
                  ]
               },
               "unitatPare":{
                  "dir3":"A99999999",
                  "dir3Pare":"A99999999",
                  "nom":"Comunidades Autonomas",
                  "codiAmbitTerritorial":"NA",
                  "nomAmbitTerritorial":"N/A",
                  "codiNivellAdministracio":"2",
                  "nomNivellAdministracio":"Administración Autonómica"
               }
            }
         }
      },
      "oficinaRegistre":{
         "dir3":"O00011864",
         "dir3UnitatResponsable":"A09018933",
         "ine10":"9821920002",
         "nom":"Consorcio Administración Abierta de Cataluña (CAOC)",
         "codiTipusVia":"2",
         "tipusVia":"Calle",
         "adressa":"Tanger",
         "numeroVia":"98",
         "codiMunicipi":"080193",
         "nomMunicipi":"Barcelona",
         "codiComarca":"13",
         "nomComarca":"Barcelonès",
         "codiProvincia":"08",
         "nomProvincia":"Barcelona",
         "codiPostal":"08018",
         "codiComunitat":"09",
         "nomComunitat":"Cataluña",
         "codiPais":"724",
         "nomPais":"España",
         "codiEntitatGeografica":"01",
         "nomEntitatGeografica":"Municipio",
         "codiNivellAdministracio":"2",
         "nomNivellAdministracio":"Administración Autonómica",
         "serveisOficina":[
            {
               "codi":"5",
               "descripcio":"Oficina Integrada en SIR"
            },
            {
               "codi":"6",
               "descripcio":"Oficina Integrada en SIR (Envío)"
            },
            {
               "codi":"7",
               "descripcio":"Oficina Integrada en SIR (Recepción)"
            }
         ],
         "unitatsServides":[
            {
               "dir3":"A09018933",
               "dir3Pare":"A09018799",
               "ine10":"9821920002",
               "nom":"Consorci Administració Oberta de Catalunya",
               "cif":"Q0801175A",
               "codiTipusVia":"2",
               "tipusVia":"Calle",
               "adressa":"Tánger",
               "numeroVia":"98",
               "codiMunicipi":"080193",
               "nomMunicipi":"Barcelona",
               "codiComarca":"13",
               "nomComarca":"Barcelonès",
               "codiProvincia":"08",
               "nomProvincia":"Barcelona",
               "codiPostal":"08018",
               "codiComunitat":"09",
               "nomComunitat":"Cataluña",
               "codiPais":"724",
               "nomPais":"España",
               "codiAmbitTerritorial":"12",
               "nomAmbitTerritorial":"Central-Autonómico",
               "codiEntitatGeografica":"01",
               "nomEntitatGeografica":"Municipio",
               "codiNivellAdministracio":"2",
               "nomNivellAdministracio":"Administración Autonómica",
               "oficinesAssociades":[
                  "O00011864"
               ]
            }
         ]
      },
      "sarcat":{
         "urPK":"0570",
         "idCentreProcedencia":34122,
         "idCentreDestinacio":44312,
         "idCentreDestinacioInterna":56334,
         "idCentreDestinacioExterna":57224
      }
   }
}
 ```

    
### 3.2.5 Bloc _Assentaments_ <a name="3.2.5"></a>

Aquest bloc conté les dades relatives als assentaments realitzats pel MUXv3_._

Els camps que formen aquest bloc es descriuen a continuació:

- **entrada** : Aquest bloc conté les dades de l'assentament d'entrada.

| **Tipus** | Bloc _TriplaAssentaments_ |
| --- | --- |

- **sortida** : Aquest bloc conté les dades de l'assentament de sortida.

| **Tipus** | Bloc _TriplaAssentaments_ |
| --- | --- |

![15](captures/15.png)

      
#### 3.2.5.1 Bloc TriplaAssentaments

Aquest bloc agrupa les dades dels assentaments realitzats, ja sigui contra el registre electrònic principal o contra l'auxiliar, així com les dades de consolidació de l'assentament realitzat en cas d'haver-se produït l'assentament al registre auxiliar.

![16](captures/16.png)

Els camps que formen aquest bloc es detallen a continuació:

- **principal** : Assentament realitzat al registre electrònic principal

| **Tipus** | Llista de bloc _Assentament_ |
| --- | --- |

Aquest bloc conté una llista amb el resultat de les execucions de la transacció MUXv3. En cas d'una execució realitzada amb èxit, el primer element de la llista contindrà les dades de l'assentament realitzat contra el registre electrònic principal, amb informació addicional per a poder realitzar un diagnòstic en cas d'error.

- **auxiliar** : Assentament realitzat al registre electrònic auxiliar

| **Tipus** | Llista de bloc _Assentament_ |
| --- | --- |

- Aquest bloc conté una llista amb el resultat de les execucions de la transacció MUXv3. En cas d'una execució realitzada amb èxit, el primer element de la llista contindrà les dades de l'assentament realitzat contra el registre electrònic auxiliar, amb informació addicional per a poder realitzar un diagnòstic en cas d'error.
- **consolidacio** : Dades de la consolidació de l'assentament auxiliar

| **Tipus** | Bloc _Consolidacio_ |
| --- | --- |

Aquest bloc conté les dades proporcionades pel servei de registre electrònic en el moment de consolidar un assentament realitzat al registre auxiliar dins del seu llibre de registre.

La consolidació és una operació que permet al registre electrònic destinatari informar a MUXv3 del número d'assentament consolidat.

Al assabentar-se el registre principal de la creació d'un assentament al registre auxiliar, en recupera les dades i el consolida al seu llibre de registre. Posteriorment, el registre electrònic pot informar mitjançant aquesta operació del número d'assentament consolidat a MUXv3.

![17](captures/17.png)

        
##### 3.2.5.1.1 Bloc Assentament

El bloc _Assentament_ conté el resultat de la execució de la transacció a MUXv3. Com es veu als apartats anteriors, el mateix bloc s'usa tant per a l'assentament d'entrada com pel assentament de sortida. Aquest bloc el formen els següents camps:

- **creation** : Instant de creació del bloc

| **Tipus** | Long |
| --- | --- |
| **valor d'exemple** | 1618994295000 |

Aquest camp conté la data i hora de creació del bloc, corresponent al moment en que s'ha executat la transacció a MUXv3. El format d'aquest camp és el conegut com a _epoch_, és a dir, el nombre de mil·lisegons transcorreguts des de l'1 de gener de 1970.

- **execucio** : Número d'execució

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 1 |

Aquest camp conté el número d'execució de la transacció MUXv3 corresponent a aquest bloc, ja que la transacció es pot executar vàries vegades fins a obtenir el resultat final (per exemple corregint el valor d'un camp no acceptat pel registre i tornant a executar).

- **resultat** : Resultat de la invocació del registre

| **Tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | true, false |

Aquest camp indica si s'ha obtingut número d'assentament o no i permet determinar de manera ràpida si l'assentament s'ha realitzat correctament o no.

- **codiError** : Codi d'error

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 54 |

Aquest camp recull el codi d'error retornat pel registre electrònic.

- **missatgeError** : Missatge informatiu

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | El valor del camp 'nom' supera la longitud màxima. |

Aquest camp conté el missatge d'error o avís retornat pel registre electrònic.

- **nomReglaEnrutament** : Nom de la regla d'enrutament aplicada

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Enrutament Ajuntament d'Arbeca |

Aquest camp conté el nom de la regla d'enrutament que ha decidit el registre cap al qual dirigir la transacció de MUXv3. La finalitat d'aquest camp és oferir als administradors del servei MUXv3 informació de diagnòstic en cas d'haver-hi problemes tècnics o errors de configuració.

- **nomRegistre** : Nom del registre al qual s'ha fet l'assentament

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Registre de l'Ajuntament d'Arbeca |

Aquest camp conté el nom que té a MUXv3 el registre al qual ha estat dirigida la transacció de MUXv3. La finalitat d'aquest camp és proporcionar als administradors del servei MUXV3 informació de diagnòstic en cas d'haver-hi problemes tècnics o errors de configuració.

- **numeroAssentament** : Número d'assentament

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | S-2021-33-00000012 |

Aquest camp conté el número d'assentament retornat pel registre electrònic.

- **dataAssentament** : Data d'assentament

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 13/04/2021 13:40:33.234+00:02 |

Aquest camp conté la data d'assentament retornada pel registre electrònic.

- **dataPresentacio** : Data de presentació

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | 01/04/2021 14:00:00.000+02:00 |

Aquest camp conté la data de presentació reconeguda pel registre electrònic.

- **evidencies** : Llista d'evidències de comunicació amb el registre electrònic

| **Tipus** | Bloc _Evidencies_ |
| --- | --- |
| **valor d'exemple** | El valor del camp 'nom' supera la longitud màxima. |

Aquest bloc conté una llista amb les evidències de comunicació entre MUXv3 i el registre electrònic. La finalitat d'aquest bloc és proporcionar informació de diagnòstic en cas d'errors així com la custòdia dels missatges originals bescanviats entre MUXv3 i el servei de registre electrònic.

![18](captures/18.png)

          
###### 3.2.5.1.1.1 Bloc _Evidencia_

El bloc _Evidencia_ conté informació relacionada amb la invocació del servei de registre electrònic per part de MUXv3. Aquest bloc el formen els següents camps:

- **creation** : Instant de creació del bloc

| **Tipus** | Long |
| --- | --- |
| **valor d'exemple** | 1618994295000 |

Aquest camp conté la data i hora de creació de la evidència, corresponent al moment en que s'ha invocat el registre electrònic. El format d'aquest camp és el conegut com a _epoch_, és a dir, el nombre de mil·lisegons transcorreguts des de l'1 de gener de 1970.

- **authorization** : Contingut de la capçalera '_Authorization_'

| **Tipus** | String |
| --- | --- |
| **valor d'exemple** | Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVC |

Aquest camp conté el contingut de la capçalera HTTP '_Authorization_'. MUXv3 usa tokens _JWT_ per a autenticar-se davant els registres electrònics que suporten les transaccions MUXv3.

- **responseHeaders** : Capçaleres HTTP de resposta

| **Tipus** | Mapa |
| --- | --- |
| **valor d'exemple** | [<br>{"Connection": "keep-alive"},<br>{"Content-Length": "17389"}<br>...<br>] |

Aquest camp conté les capçaleres HTTP rebudes amb la resposta del registre electrònic.

- **request** : Petició enviada al registre electrònic

| **Tipus** | Byte[] (Base64) |
| --- | --- |
| **valor d'exemple** | eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCeyJhbGciOiJIUzI1NiIsIn... |

Aquest camp conté la petició tal qual ha estat enviada cap al registre electrònic. Per a reduir el volum de la evidència, la petició ha estat prèviament comprimida amb GZIP.

- **response** : Resposta rebuda del registre electrònic

| **Tipus** | Byte[] (Base64) |
| --- | --- |
| **valor d'exemple** | eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCeyJhbGciOiJIUzI1NiIsIn... |

Aquest camp conté la resposta tal qual ha estat rebuda del registre electrònic. Per a reduir el volum de la evidència, la petició ha estat prèviament comprimida amb GZIP.

- **connectTimedOut** : Flag de timeout de connexió

| **Tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | false |

Aquest camp indica si s'ha produït un _timeout_ (exhauriment del temps màxim d'espera) en la connexió amb el registre electrònic. El valor _true_ voldrà dir que no ha estat possible ni tant sols connectar amb el registre pel que la petició no ha estat rebuda pel registre.

- **readTimedOut** : Flag de timeout de lectura

| **Tipus** | Boolean |
| --- | --- |
| **valor d'exemple** | True |

Aquest camp indica si s'ha produït un _timeout_ (exhauriment del temps màxim d'espera) en la recepció de la resposta del registre electrònic. Un _timeout_ de lectura implica que la petició segurament hagi estat rebuda pel registre i que aquesta acabarà sent processada, tot i que no s'ha pogut obtenir la resposta.

- **elapsedTime** : Temps d'invocació del registre

| **Tipus** | Integer |
| --- | --- |
| **valor d'exemple** | 235 |

Aquest camp conté el temps total, en mil·lisegons, que ha pres la invocació del registre electrònic.

A continuació és mostra un exemple de bloc _Assentaments_

```json
"assentaments": {
  "entrada": {
    "principal": [
      {
        "creation": 1619416531407,
        "execucio": 1,
        "resultat": false,
        "missatgeError": "S'ha exhaurit el temps màxim d'espera connectant amb el servei",
        "nomReglaEnrutament": "Regla enrutament Ajuntament de Masquefa",
        "nomRegistre": "Registre de l’Ajuntament de Masquefa",
        "evidencies": [
          {
            "creation": 1619416532412,
            "request": "H4sIAAAAAAAAAI1U32/...",
            "connectTimedOut": true,
            "readTimedOut": false,
            "elapsedTime": 1000
          }
        ]
      }
    ],
    "auxiliar": [
      {
        "creation": 1619416532407,
        "execucio": 1,
        "resultat": true,
        "codiError": 0,
        "nomReglaEnrutament": "Regla per defecte",
        "nomRegistre": "[DEV] EACAT",
        "numeroAssentament": "E/000345-2021",
        "dataAssentament": "2021-04-26T07:55:55+02:00",
        "dataPresentacio": "2021-04-26T07:55:55+02:00",
        "evidencies": [
          {
            "creation": 1619416583523,
            "responseHeaders": {
              "date": "Mon, 26 Apr 2021 05:55:55 GMT",
              "_result": "HTTP/1.1 200 OK",
              "server": "Microsoft-IIS/7.5",
              "content-length": "2232",
              "keep-alive": "timeout=10, max=100",
              "x-aspnet-version": "4.0.30319",
              "x-powered-by": "ASP.NET",
              "connection": "Keep-Alive",
              "content-type": "text/xml; charset=utf-8",
              "cache-control": "private, max-age=0"
            },
            "request": "H4sIAAAAAAAAAI1U32/...",
            "response": "H4sIAAAAAAAAAL1WXW/...",
            "connectTimedOut": false,
            "readTimedOut": false,
            "elapsedTime": 253
          }
        ]
      }
    ]
  },
  "sortida": {
    "principal": [
      {
        "creation": 1619416532407,
        "execucio": 1,
        "resultat": true,
        "codiError": 0,
        "nomReglaEnrutament": "Regla per defecte",
        "nomRegistre": "[DEV] EACAT",
        "numeroAssentament": "S/000153-2021",
        "dataAssentament": "2021-04-26T07:55:50+02:00",
        "dataPresentacio": "2021-04-26T07:55:50+02:00",
        "evidencies": [
          {
            "creation": 1619416532412,
            "responseHeaders": {
              "date": "Mon, 26 Apr 2021 05:55:50 GMT",
              "_result": "HTTP/1.1 200 OK",
              "server": "Microsoft-IIS/7.5",
              "content-length": "2232",
              "keep-alive": "timeout=10, max=100",
              "x-aspnet-version": "4.0.30319",
              "x-powered-by": "ASP.NET",
              "connection": "Keep-Alive",
              "content-type": "text/xml; charset=utf-8",
              "cache-control": "private, max-age=0"
            },
            "request": "H4sIAAAAAAAAAI1U32/...",
            "response": "H4sIAAAAAAAAAL1WXW/...",
            "connectTimedOut": false,
            "readTimedOut": false,
            "elapsedTime": 387
          }
        ]
      }
    ]
  }
}
```

    
### 3.2.6 Bloc _Rebuts_

Aquest bloc conté els identificadors dels documents (UUID) que constitueixen els rebuts o justificants (en format PDF) generats pel servei o aplicació que llança l'assentament.

Aquests documents de rebut son generats amb posterioritat a la realització de l'assentament, pel que aquesta informació no viatjarà dins de la petició cap al registre electrònic, si no que quedarà registrada a la transacció per poder ser consultada amb posterioritat pel registre.

![19](captures/19.png)

_1 - La aplicació executa la transacció i es crea el nou assentament._

_2 - La aplicació genera el document de rebut i el carrega a DESA'L._

_3 - La aplicació adjunta el rebut a MUX._

_4 - El registre comprova contra MUX si s'ha adjuntat el rebut a l'assentament._

_5 - El registre descarrega el debut de DESA'L._

Els camps que formen aquest bloc es descriuen a continuació:

- **rebutEntrada** : Identificador del document de rebut d'entrada.

| **Tipus** | String |
| --- | --- |
| **Valor d'exemple** | ba51a9be-ad53-4fc4-ba90-53355456e2cf |

Aquest camp conté l'UUID del document de DESA'L corresponent al rebut de l'assentament d'entrada, és a dir, el justificant que s'entrega o s'envia per correu al ciutadà o usuari que desencadena la creació de l'assentament.

- **rebutSortida** : Identificador del document de rebut de sortida.

| **Tipus** | String |
| --- | --- |
| **Valor d'exemple** | 93cbf258-593d-4c84-bf12-1abd501bce91 |

Aquest camp conté l'UUID del document de DESA'L corresponent al rebut de l'assentament de sortida, és a dir, el justificant que s'entrega o s'envia per correu al ciutadà o usuari que desencadena la creació de l'assentament.

   
### 3.2.7 Bloc _DadesLliures_

Aquest bloc està format per un _map_ que permet modelar un document dintre del propi bloc. Així, una aplicació pot transmetre informació addicional, particular al seu negoci, cap al registre electrònic.

![20](captures/20.png)

Per exemple, un servei de factura electrònica pot posar dintre d'aquest bloc la factura electrònica que s'ha registrat.

```json
{
  "dadesLliures": {
    "FileHeader": {
      "SchemaVersion": "3.1",
      "Modality": "I",
      "InvoiceIssuerType": "EM",
      "Batch": {
        "BatchIdentifier": "A99887723392009-",
        "InvoicesCount": "1",
        "TotalInvoicesAmount": {
          "TotalAmount": 468.10
        },
        "TotalOutstandingAmount": {
          "TotalAmount": 468.10
        },
        "TotalExecutableAmount": {
          "TotalAmount": 489.74
        },
        "InvoiceCurrencyCode": "EUR"
      }
    },
    "Parties": {
      "SellerParty": {
        "TaxIdentification": {
          "PersonTypeCode": "J",
          "ResidenceTypeCode": "R",
          "TaxIdentificationNumber": "A82735122"
        },
        "LegalEntity": {
          "CorporateName": "Company Comp SA",
          "TradeName": "Comp",
          "RegistrationData": {
            "Book": "1",
            "RegisterOfCompaniesLocation": "12AP22",
            "Sheet": "3",
            "Folio": "15",
            "Section": "2",
            "Volume": "12",
            "AdditionalRegistrationData": "Sin datos"
          },
          "AddressInSpain": {
            "Address": "C/ Mayour 33 15º E",
            "PostCode": "28001",
            "Town": "Argamasilla de Alba",
            "Province": "Ciudad Real",
            "CountryCode": "ESP"
          },
          "ContactDetails": {
            "Telephone": "917776665",
            "TeleFax": "917776666",
            "WebAddress": "www.facturae.es",
            "ElectronicMail": "facturae@mityc.es",
            "ContactPersons": "Fernando",
            "CnoCnae": "28000",
            "INETownCode": "2134AAB",
            "AdditionalContactDetails": "Otros datos"
          }
        }
      },
      "BuyerParty": {
        "TaxIdentification": {
          "PersonTypeCode": "J",
          "ResidenceTypeCode": "R",
          "TaxIdentificationNumber": "A99887723"
        },
        "LegalEntity": {
          "CorporateName": "Bakgammon Games",
          "AddressInSpain": {
            "Address": "C/ Mayor 211",
            "PostCode": "28001",
            "Town": "Madrid",
            "Province": "Madrid",
            "CountryCode": "ESP"
          },
          "ContactDetails": {
            "Telephone": "91562553",
            "TeleFax": "91562554",
            "ContactPersons": "Manolo y Juanito",
            "INETownCode": "0199op"
          }
        }
      }
    },
    "Invoices": {
      "Invoice": {
        "InvoiceHeader": {
          "InvoiceNumber": "39",
          "InvoiceSeriesCode": "2009-",
          "InvoiceDocumentType": "FC",
          "InvoiceClass": "OO"
        },
        "InvoiceIssueData": {
          "IssueDate": "2009-02-25",
          "OperationDate": "2009-02-24",
          "PlaceOfIssue": {
            "PostCode": "28001",
            "PlaceOfIssueDescription": "Argamasilla de Alba"
          },
          "InvoicingPeriod": {
            "StartDate": "2009-02-20",
            "EndDate": "2009-02-25"
          },
          "InvoiceCurrencyCode": "EUR",
          "TaxCurrencyCode": "EUR",
          "LanguageName": "es"
        },
        "TaxesOutputs": {
          "Tax": {
            "TaxTypeCode": "01",
            "TaxRate": 16.00,
            "TaxableBase": {
              "TotalAmount": 441.60
            },
            "TaxAmount": {
              "TotalAmount": 70.66
            }
          }
        },
        "TaxesWithheld": {
          "Tax": {
            "TaxTypeCode": "04",
            "TaxRate": 15.00,
            "TaxableBase": {
              "TotalAmount": 441.60
            },
            "TaxAmount": {
              "TotalAmount": 66.24
            }
          }
        },
        "InvoiceTotals": {
          "TotalGrossAmount": 441.60,
          "GeneralDiscounts": {
            "Discount": {
              "DiscountReason": "Descuento global",
              "DiscountRate": 10.0000,
              "DiscountAmount": 44.16
            }
          },
          "GeneralSurcharges": {
            "Charge": {
              "ChargeReason": "Gastos envío",
              "ChargeRate": 15.0000,
              "ChargeAmount": 66.24
            }
          },
          "TotalGeneralDiscounts": 44.16,
          "TotalGeneralSurcharges": 66.24,
          "TotalGrossAmountBeforeTaxes": 463.68,
          "TotalTaxOutputs": 70.66,
          "TotalTaxesWithheld": 66.24,
          "InvoiceTotal": 468.10,
          "ReimbursableExpenses": {
            "ReimbursableExpenses": {
              "ReimbursableExpensesSellerParty": {
                "PersonTypeCode": "F",
                "ResidenceTypeCode": "E",
                "TaxIdentificationNumber": "A33445556"
              },
              "ReimbursableExpensesBuyerParty": {
                "PersonTypeCode": "J",
                "ResidenceTypeCode": "U",
                "TaxIdentificationNumber": "A0009283"
              },
              "ReimbursableExpensesAmount": 20.99
            }
          },
          "TotalFinancialExpenses": 2.99,
          "TotalOutstandingAmount": 468.10,
          "AmountsWithheld": {
            "WithholdingReason": "Retencion",
            "WithholdingRate": 0.5000,
            "WithholdingAmount": 2.34
          },
          "TotalExecutableAmount": 489.74,
          "TotalReimbursableExpenses": 20.99
        },
        "Items": {
          "InvoiceLine": {
            "IssuerContractReference": "1234A",
            "IssuerTransactionReference": "1234B",
            "ReceiverContractReference": "1234C",
            "ReceiverTransactionReference": "1234D",
            "PurchaseOrderReference": “123.00”,
            "DeliveryNotesReferences": {
              "DeliveryNote": {
                "DeliveryNoteNumber": "444ALB-234"
              }
            },
            "ItemDescription": "Miniordenador",
            "Quantity": 2.00,
            "UnitOfMeasure": "01",
            "UnitPriceWithoutTax": 230.000000,
            "TotalCost": 460.00,
            "DiscountsAndRebates": {
              "Discount": {
                "DiscountReason": "Descuento a empresas",
                "DiscountRate": 5.0000,
                "DiscountAmount": 23.00
              }
            },
            "Charges": {
              "Charge": {
                "ChargeReason": "Gastos de transporte",
                "ChargeRate": 1.0000,
                "ChargeAmount": 4.60
              }
            },
            "GrossAmount": 441.60,
            "TaxesWithheld": {
              "Tax": {
                "TaxTypeCode": "04",
                "TaxRate": 15.00,
                "TaxableBase": {
                  "TotalAmount": 441.60
                },
                "TaxAmount": {
                  "TotalAmount": 66.24
                }
              }
            },
            "TaxesOutputs": {
              "Tax": {
                "TaxTypeCode": "01",
                "TaxRate": 16.00,
                "TaxableBase": {
                  "TotalAmount": 441.60
                },
                "TaxAmount": {
                  "TotalAmount": 70.66
                },
                "EquivalenceSurcharge": 0.00,
                "EquivalenceSurchargeAmount": {
                  "TotalAmount": 0.00
                }
              }
            },
            "AdditionalLineItemInformation": "Miniordenadores marca H"
          }
        },
        "PaymentDetails": {
          "Installment": {
            "InstallmentDueDate": "2009-02-25",
            "InstallmentAmount": 400.98,
            "PaymentMeans": "02",
            "AccountToBeCredited": {
              "IBAN": "30000098347",
              "BankCode": "1112",
              "BranchCode": "0098",
              "BranchInSpainAddress": {
                "Address": "C/ Francia 24",
                "PostCode": "28010",
                "Town": "Madrid",
                "Province": "Madrid",
                "CountryCode": "ESP"
              }
            },
            "PaymentReconciliationReference": "1234-txxe",
            "AccountToBeDebited": {
              "IBAN": "12340001234",
              "BankCode": "1234",
              "BranchCode": "37",
              "BranchInSpainAddress": {
                "Address": "C/ Palomo 22",
                "PostCode": "28000",
                "Town": "Madrid",
                "Province": "Madrid",
                "CountryCode": "ESP"
              }
            }
          }
        },
        "LegalLiterals": [
          "Donaciones",
          "Subvenciones"
        ],
        "AdditionalData": {
          "RelatedDocuments": {
            "Attachment": {
              "AttachmentCompressionAlgorithm": "NONE",
              "AttachmentFormat": "gif",
              "AttachmentEncoding": "BASE64",
              "AttachmentDescription": "Icono de un correo",
              "AttachmentData": "R0lGODlhJQAtAMZFAACn5uHh+..."
            }
          }
        }
      }
    }
  }
}
```

# 4 Operacions <a name="4"></a>

## 4.1 Operacions disponibles <a name="4.1"></a>

MUX V3 ofereix les operacions que es resumeixen a la següent taula:

| Mètode HTTP | Nom de la operació | |
| --- | --- | --- |
| **TRANSACCIO** | - |**REGISTRE** |
| **GET** | Consulta de la transacció. | ✔ | 
| **OPERACIONS DE BUSTIA D'ASSENTAMENTS AUXILIARS** |- | -| 
| **GET** | Llegir transacció ingressada al registre auxiliar | ✔ |
| **GET** | Avançar el cursor a la següent transacció | ✔ |
| **GET** | Retrocedir el cursor a la transacció anterior | ✔ |
| **GET** | Posicionar el cursor a una transacció en particular | ✔ |
| **GET** | Consultar la posició del cursor | ✔ |
| **POST** | Compactar la bústia a partir d'una transacció determinada | ✔ |
| **OPERACIONS DE CONSOLIDACIÓ D'ASSENTAMENTS** |- | -| -|
| **GET** | Recuperació de les dades de consolidació dels assentaments auxiliars. | ✔ |
| **POST** | Establiment de les dades de consolidació dels assentaments auxiliars. | ✔ |
| **DELETE** | Eliminació de les dades de consolidació dels assentaments auxiliars. | ✔ |




### 4.1.1 Operacions d'establiment dels rebuts/justificants

#### 4.1.1.1 Consulta dels rebuts associats a la transacció

| GET https://mux-dev.aoc.cat/transaccio/ **{{idTransaccio}}** /rebuts/ **{{sentit}}** |
| --- |

Aquesta operació permet recuperar els identificadors de document corresponents als rebuts o justificants generats per la aplicació i carregats a DESA'L després d'haver creat l'assentament.

En el cas dels documents de rebut o justificant d'assentament, MUXv3 no incorpora a la transacció les metadades dels documents que puguin estar registrades a DESA'L.

**Petició**

- **idTransaccio** : Identificador de la transacció de la qual es vol consultar els rebuts o justificants.
- **sentit** : Paràmetre opcional. Tipus d'assentament per al qual es vol consultar el rebut o justificant d'assentament.

Al existir les transaccions duals o interadministratives, pot donar-se el cas en que una transacció té dos rebuts associats, un per a l'assentament d'entrada i un per a l'assentament de sortida. Així, de no informar-se aquest paràmetre MUXv3 retornarà la informació de tots els rebuts que hi hagi associats a la transacció.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

- **errors** : Llista de cadenes de text amb la descripció dels errors que s'hagin pogut produir durant la execució de la operació.

- **idTransaccio** : Identificador de la transacció. El format de l'identificador de la transacció és un número de 64 bits expressat en hexadecimal, en quatre grups: xxxx-xxxx-xxxx-xxxx. Per exemple 0b9f-bf17-e8c1-b031.

- **rebuts** : Bloc que conté els identificadors dels documents de rebut o justificants custodiats a DESA'L. Aquest bloc conté els dos següents camps:

  - **rebutSortida** : Identificador en format UUID del document de DESA'L corresponent al rebut de l'assentament de sortida.
  - **rebutEntrada** : Identificador en format UUID del document de DESA'L corresponent al rebut de l'assentament d'entrada.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició 
```json
curl -X GET https://mux-dev.aoc.cat/transaccio/0bac-9d42-6841-303b/rebuts 
```
Resposta
```json
{
   "resultat":"ok",
   "idTransaccio":"0bac-9d42-6841-303b",
   "rebuts":{
      "rebutSortida":"93cbf258-593d-4c84-bf12-1abd501bce91",
      "rebutEntrada":"29fa3341-02d1-6934-fa04-3db1783cfa23"
   }
}
 ```

Petició
```json
 curl -X GET https://mux-dev.aoc.cat/transaccio/0bac-9d42-6841-303b/rebuts/E 
 ```

 Resposta
 ```json
{
   "resultat":"ok",
   "idTransaccio":"0bac-9d42-6841-303b",
   "rebuts":{
      "rebutEntrada":"29fa3341-02d1-6934-fa04-3db1783cfa23"
   }
}
 ```
 Petició 
 ```json
  curl -X GET https://mux-dev.aoc.cat/transaccio/0bac-9d42-6841-303b/rebuts/S 
```
 Resposta 
 ```json
{
   "resultat":"ko",
   "errors":[
      "No s'ha trobat el rebut de l'assentament en sentit 'E'=Entrada per a la transacció amb id='0bac-9d42-6841-303b'"
   ],
   "idTransaccio":"0bac-9d42-6841-303b"
}
 ```

 
### 4.1.2 Operacions de bústia d'assentaments realitzats al registre auxiliar

MUXv3 permet configurar un registre electrònic auxiliar que substituirà el registre titular davant eventuals problemes tècnics, de comunicacions o intervencions per manteniment d'aquest.

Una funcionalitat imprescindible a MUXv3 és dotar el registre electrònic titular d'un mecanisme eficient per a recuperar la informació relativa a les transaccions que han estat registrades al registre auxiliar. Amb aquesta finalitat s'ha implementat a MUXv3 un sistema de bústies a on es podrà llegir totes aquelles transaccions que han estat registrades al registre auxiliar.

Cada registre electrònic tindrà assignat un identificador de bústia que serà generat en el moment de donar d'alta el registre a MUXv3. Posteriorment es podrà accedir al servei de bústia mitjançant un conjunt d'operacions molt simples que es descriuen dins d'aquest apartat del document.

Les bústies venen a ser una mena de llista d'elements amb un cursor que permet anar navegant per aquesta, ja sigui avançant cap al final de la llista (l'element més recent) o retrocedint el cursor per a tornar a llegir una transacció que no es va poder consolidar correctament.

![25](captures/25.png)

La mecànica suggerida per a consultar la bústia d'assentaments s'il·lustra al següent gràfic:

![26](captures/26.png)

- _Següent_ : Si la bústia no té elements nous, es retornarà el codi de resultat 'buida' i la operació de verificació de la bústia haurà finalitzat. En cas contrari es rebrà el codi '_ok_' i el cursor de lectura apuntarà al nou element disponible.
- _Llegir_ : La operació de lectura de l'element actual a la bústia retornarà la transacció de MUXv3 completa, incloent la informació de l'assentament realitzat al registre auxiliar.
- _Processar_ : A aquesta etapa el registre electrònic processarà la transacció i realitzarà les accions que consideri oportunes.

Davant la possibilitat de que es produeixin situacions en les quals sigui necessari tornar a processar alguns elements de la bústia, existeixen operacions auxiliars que permeten situar el cursor de lectura a un element en particular a partir del qual es vulgui tornar a realitzar tot el processament.

![image](https://user-images.githubusercontent.com/32306731/137281698-9dfc2044-94f7-487f-a7d6-9a4e0707feb3.png) Les bústies no suporten l'execució concurrent de les operacions. És a dir, donada una bústia amb identificador X totes les operacions llançades sobre aquesta s'executaran de manera seqüencial, fent esperar la execució de la darrera operació rebuda fins a que les operacions anteriors hagin finalitzat la seva execució.Aquesta limitació no aplica entre bústies diferents. 


#### 4.1.2.1 Llegir transacció de la bústia

| GET https://mux-dev.aoc.cat/bustia/{{idBustia}}/llegir |
| --- |

Aquesta operació retorna la transacció corresponent a l'element al qual apunta el cursor de lectura de la bústia.

**Petició**

- **idBustia** : Identificador de la bústia de la qual es vol llegir.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple avançar al següent element per a una bústia que està buida), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme l'operació.

- **element** : Conté la posició que ocupa l'element dintre de la bústia.

- **idTransaccio** : Identificador de la transacció MUXv3.

- **transaccio** : Bloc que conté totes les dades de la transacció, tal qual les rebria el registre electrònic al ser invocat pel MUXv3. En aquest cas, la transacció incorpora també les dades de l'assentament realitzat al registre auxiliar.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició 
```json 
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/llegir 
```

Resposta 
```json
{
  "resultat": "ok",
  "element": 11,
  "idTransaccio": "0ba6-4bca-0541-9039",
  "transaccio": {
    "idTransaccio": "0ba6-4bca-0541-9039",
    "execucions": 1,
    "tipus": "S",
    "dataCreacio": 1620208900118,
    "dataDarreraExecucio": 1620208910315,
    "control": {
      "aplicacio": "INTEGRADOR-MUX",
      "urlAplicacio": "https://www.aoc.cat",
      "observacionsApunt": "Observacions sobre l'apunt a realitzar ",
      ...
    },
    "assumpte": {
      "resum": "Objecte de l'assentament de sortida",
      "observacions": "Observacions a precisar relatives a l'assentament",
      "dataPresentacio": 1615907378000,
      "codiAssumpteSegonsDesti": "0000000001",
      ...
    },
    "registres": {
      "origen": {
        "unitatOrganica": {
          "dir3": "A09018933",
          "dir3Pare": "A09018799",
          "ine10": "9821920002",
          "nom": "Consorci Administració Oberta de Catalunya",
          "cif": "Q0801175A",
          ...
        }
      }
    },
    "interessats": [
      ...
    ],
    "documents": {
      "ba51a9be-ad53-4fc4-ba90-53355456e2cf": {
        ...
      },
      "93cbf258-593d-4c84-bf12-1abd501bce91": {
        ...
      }
    },
    "assentaments": {
      "sortida": {
        "auxiliar": [
          {
            "creation": 1620208910390,
            "execucio": 1,
            "resultat": true,
            "codiError": 0,
            "nomReglaEnrutament": "Regla per defecte",
            "nomRegistre": "[DEV] EACAT",
            "numeroAssentament": "S/000160-2021",
            "dataAssentament": "2021-05-05T12:02:22+02:00",
            "dataPresentacio": "2021-05-05T12:02:22+02:00",
            "evidencies": [
              {
                "creation": 1620208910394,
                "responseHeaders": {
                  "date": "Wed, 05 May 2021 10:02:24 GMT",
                  "_result": "HTTP/1.1 200 OK",
                  "server": "Microsoft-IIS/7.5",
                  "content-length": "2232",
                  "keep-alive": "timeout=10, max=100",
                  "x-aspnet-version": "4.0.30319",
                  "x-powered-by": "ASP.NET",
                  "connection": "Keep-Alive",
                  "content-type": "text/xml; charset=utf-8",
                  "cache-control": "private, max-age=0"
                },
                "request": "H4sIAAAAAAAAAI1...",
                "response": "H4sIAAAAAAAAAL1...",
                "connectTimedOut": false,
                "readTimedOut": false,
                "elapsedTime": 3702
              }
            ]
          }
        ]
      }
    },
    "instanciaGenerica": {
      ...
    },
    "ttl": 20
  }
}
 ```

Petició
```json
 curl –X GET https://mux-dev.aoc.cat/bustia/AA2D3156271D/llegir 
 ```


 Resposta
 ```json
 {"resultat": "ko","missatge": "No existeix cap bústia amb aquest identificador"} 
 ```

#### 4.1.2.2 Avançar el cursor a la següent transacció

| GET https://mux-dev.aoc.cat/bustia/{{idBustia}}/seguent |
| --- |

Aquesta operació fa avançar una posició el cursor de lectura de la bústia, cap a un possible element més recent.

**Petició**

- **idBustia** : Identificador de la bústia de la qual es vol fer avançar el cursor.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple avançar al següent element per a una bústia que està buida), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme l'operació.

- **element** : Conté la posició en la qual es troba el cursor de lectura.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/seguent
```
Resposta
```json
{
   "resultat":"ok",
   "element":10
}
 ```
Petició
```json
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/seguent
```
Resposta
```json
{
   "resultat":"buida",
   "missatge":"No hi ha elements nous a la bústia",
   "element":11
}
```

 #### 4.1.2.3 Retrocedir el cursor a la transacció anterior

| GET https://mux-dev.aoc.cat/bustia/{{idBustia}}/anterior |
| --- |

Aquesta operació fa retrocedir una posició el cursor de lectura de la bústia, cap a un element anterior a l'actual.

**Petició**

- **idBustia** : Identificador de la bústia de la qual es vol fer retrocedir el cursor.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple avançar al següent element per a una bústia que està buida), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme l'operació.

- **element** : Conté la posició en la qual es troba el cursor de lectura.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició 
```json
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/anterior 
```

Resposta
```json
{"resultat": "ok","element": 8} 
```
Petició
```json
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/anterior
```

Resposta
```json
{"resultat": "ko","missatge": "El primer element disponible a la bústia es troba a la posició 8","element": 8}
```

#### 4.1.2.4 Posicionar el cursor a una transacció en particular

| GET https://mux-dev.aoc.cat/bustia/ **{{idBustia}}** /situar/ **{{posicio}}** |
| --- |

Aquesta operació estableix la posició del cursor de lectura a un element en concret.

**Petició**

- **idBustia** : Identificador de la bústia de la qual es vol situar el el cursor.
- **posicio** : Número de l'element al qual es vol situar el cursor.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple situar el cursor a una posició inexistent), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme aquesta.

- **element** : Conté la posició en la qual es troba el cursor de lectura.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
 curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/situar/9 
 ```
Resposta
```json
{
   "resultat":"ok",
   "element":9
}
 ```
Petició
```json
curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/situar/2000 
```
Resposta
```json
{
   "resultat":"ko",
   "missatge":"No es pot situar el punter de lectura a la posició indicada. El nombre màxim d'element és el 11",
   "element":9
}
 ``` 

#### 4.1.2.5 Consultar la posició del cursor

| GET https://mux-dev.aoc.cat/bustia/ **{{idBustia}}** /posicio |
| --- |

Aquesta operació consulta la posició actual del cursor de lectura.

**Petició**

- **idBustia** : Identificador de la bústia de la qual es vol situar el el cursor.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple situar el cursor a una posició inexistent), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme aquesta.

- **element** : Conté la posició en la qual es troba el cursor de lectura.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
´´´json
 curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/posicio
 ´´´

Resposta
```json
{
   "resultat":"ok",
   "element":8
}
 ```

Petició
```json
 curl –X GET https://mux-dev.aoc.cat/bustia/083AF3241D28/posicio 
 ``
Resposta
```json
{
   "resultat":"ko",
   "missatge":"No existeix cap bústia amb aquest identificador"
}
 ```

#### 4.1.2.6 Compactar la bústia a partir d'una transacció determinada

Amb el temps s'aniran acumulant elements a la bústia fent que aquesta arribi a ocupar un gran espai d'emmagatzemament, factor que pot fer que el rendiment del servei de bústia sigui pitjor i fins i tot ocasionar problemes d'estabilitat a la plataforma.

Per tal de poder mantenir un consum d'espai d'emmagatzemament raonable sense perjudicar el funcionament del servei de bústia existeix una operació de compactació de la bústia, que permet eliminar d'aquesta tots els elements a una posició inferior a la passada per paràmetre a aquesta operació.

![27](captures/27.png)

| POST https://mux-dev.aoc.cat/bustia/ **{{idBustia}}** /compactar/ **{{posicio}}** |
| --- |

Aquesta operació consulta la posició actual del cursor de lectura.

**Petició**

- **idBustia** : Identificador de la bústia que es vol compactar.
- **posicio** : Número d'element a partir del qual es vol compactar (inclòs).

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament.

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

**buida** : Si no hi ha elements nous a la bústia.

- **missatge** : En cas d'error conté la descripció de l'error que s'ha produït o bé, si la bústia es troba en un estat que no permet realitzar alguna operació (per exemple situar el cursor a una posició inexistent), conté un missatge que descriu el motiu pel qual no s'ha pogut dur a terme aquesta.

- **element** : Conté la posició a la qual s'ha compactat la bústia. En aquest cas no és retorna lal posició actual del cursor de lectura.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
curl –X POST https://mux-dev.aoc.cat/bustia/083AF3241D28/compactar/9 
```


Resposta
```json
{
   "resultat":"ok",
   "element":9
}
 ```
Petició
```json
curl –X POST https://mux-dev.aoc.cat/bustia/083AF3241D28/compactar/20 
```
Resposta
```json
{
   "resultat":"ko",
   "missatge":"S'ha produit un error tècnic servint aquesta petició.",
   "element":20
}
 ```

### 4.1.3 Operacions de consolidació d'assentaments

#### 4.1.3.1 Recuperació de les dades de consolidació dels assentaments auxiliars

| GET https://mux-dev.aoc.cat/transaccio/ **{{idTransaccio}}** /consolidacio |
| --- |

Aquesta operació retorna les dades dels assentaments creats per la transacció, que inclouen, d'haver-se realitzat, les dades de consolidació dels assentaments realitzats al registre auxiliar.

**Petició**

- **idTransaccio** : Identificador de transacció per a la qual es vol consultar les dades de consolidació.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

- **errors** : Llista de cadenes de text amb la descripció dels errors que s'hagin pogut produir durant la execució de la operació.

- **idTransaccio** : Identificador de la transacció. El format de l'identificador de la transacció és un número de 64 bits expressat en hexadecimal, en quatre grups: xxxx-xxxx-xxxx-xxxx. Per exemple 0b9f-bf17-e8c1-b031.

- **assentaments** : Bloc _Assentaments_ (veure secció 3.2.5 d'aquest document) amb la informació dels assentaments realitzats a la transacció.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
 curl –X GET https://mux-dev.aoc.cat/transaccio/0bad-f45b-b743-f000/consolidacio 
 ```

Resposta
```json
{
  "resultat": "ok",
  "idTransaccio": "0bad-f45b-b743-f000",
  "tipusTransaccio": "E",
  "ttlRestantEnSegons": 1043,
  "assentaments": {
    "entrada": {
      "auxiliar": [
        {
          "creation": 1620722995145,
          "execucio": 1,
          "resultat": true,
          "codiError": 0,
          "nomReglaEnrutament": "ERES Consorci AOC",
          "nomRegistre": "[DEV] ERES CAOC V2 (Audifilm)",
          "idBustia": "70446C69F6AC",
          "numeroAssentament": "E2021000626",
          "dataAssentament": "2021-05-11T10:49:56.000+02:00",
          "dataPresentacio": "2021-05-11T10:47:41.000+02:00",
          "evidencies": [
            {
              "creation": 1620722995344,
              "responseHeaders": {
                "date": "Tue, 11 May 2021 08:49:56 GMT",
                "_result": "HTTP/1.1 200 OK",
                "server": "Apache",
                "transfer-encoding": "chunked",
                "keep-alive": "timeout=10, max=100",
                "connection": "Keep-Alive",
                "content-type": "multipart/related;start=\"<rootpart*c7ec123e-a155-49e4-8fa2-e4e3e19d7832@example.jaxws.sun.com>\";type=\"application/xop+xml\";boundary=\"uuid:c7ec123e-a155-49e4-8fa2-e4e3e19d7832\";start-info=\"text/xml\"",
                "cache-control": "no-cache"
              },
              "request": "H4sIAAAAAAAA",
              "response": "H4sIAAAAAAAA",
              "connectTimedOut": false,
              "readTimedOut": false,
              "elapsedTime": 1419
            }
          ]
        }
      ]
    }
  }
}
 ```

Petició
```json
curl –X GET https://mux-dev.aoc.cat/transaccio/0bad-f45b-b743-f000/consolidacio
```

Resposta
```json
{"resultat": "ko","errors": ["La transacció no ha estat executada o no ha obtingut cap número d'assentament"]}
```

#### 4.1.3.2 Establiment de les dades de consolidació dels assentaments auxiliars

| POST https://mux-dev.aoc.cat/transaccio/ **{{idTransaccio}}** /consolidacio |
| --- |

Aquesta operació estableix les dades de consolidació per als assentaments creats al registre auxiliar per a la transacció.

**Petició**

- **idTransaccio** : Identificador de transacció per a la qual es vol establir les dades de consolidació.

**Cos de la petició (JSON)**

El cos de la petició està format pels següents camps:

- **sentit** : Tipus d'assentament, entrada o sortida, que es vol consolidar. Aquest paràmetre aplica sobretot a les transaccions duals o interadministratives.
- **numeroAssentamentPerConsolidar** : Número d'assentament donat pel registre auxiliar. Com a mecanisme de seguretat es demana el número d'assentament auxiliar per a minimitzar la possibilitat de que es pugui consolidar per equivocació un assentament incorrecte.
- **numeroAssentament** : Número d'assentament emès pel registre titular i contra el qual s'està consolidant l'assentament auxiliar.
- **dataAssentament** : Data d'assentament emesa pel registre principal amb el format de Timestamp [dd/mm/aaaa hh:mi:SS.sss+zz:zz]. Per exemple:</b>01/02/2019 11:48:00.342+00:01.
- **dataPresentacio** : Data de presentació reconeguda pel registre titular amb el format de Timestamp [dd/mm/aaaa hh:mi:SS.sss+zz:zz]. Per exemple:</b>01/02/2019 11:48:00.342+00:01.
- **evidencia** : Binari en format Base64 amb un document que demostra l'origen de l'assentament i per tant garanteix el no-repudi de l'assentament emès. Tot i no ser crític i poder-se ometre, és recomanable informar sempre aquest element.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

- **errors** : Llista de cadenes de text amb la descripció dels errors que s'hagin pogut produir durant la execució de la operació.

- **idTransaccio** : Identificador de la transacció. El format de l'identificador de la transacció és un número de 64 bits expressat en hexadecimal, en quatre grups: xxxx-xxxx-xxxx-xxxx. Per exemple 0b9f-bf17-e8c1-b031.

- **assentaments** : Bloc _Assentaments_ (veure secció 3.2.5 d'aquest document) amb la informació dels assentaments realitzats a la transacció.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
 curl –X POST https://mux-dev.aoc.cat/transaccio/0bad-f45b-b743-f000/consolidacio 
 ```

Cos de la petició 
```json
{
	"sentit": "E",
	"numeroAssentamentPerConsolidar" : "E2021000626",
	"numeroAssentament": "E-00002256-2021",
	"dataAssentament": "01/02/2019 11:48:00.342+00:01",
	"dataPresentacio": "01/02/2019 11:48:00.342+00:01",
	"evidencia": "ICAgIDxtYXAgb..."
}

 ```

Resposta
```json
{
  "resultat": "ok",
  "idTransaccio": "0bad-f45b-b743-f000",
  "tipusTransaccio": "E",
  "ttlRestantEnSegons": 1043,
  "assentaments": {
    "entrada": {
      "auxiliar": [
        {
          "creation": 1620722995145,
          "execucio": 1,
          "resultat": true,
          "codiError": 0,
          "nomReglaEnrutament": "ERES Consorci AOC",
          "nomRegistre": "[DEV] ERES CAOC V2 (Audifilm)",
          "idBustia": "70446C69F6AC",
          "numeroAssentament": "E2021000626",
          "dataAssentament": "2021-05-11T10:49:56.000+02:00",
          "dataPresentacio": "2021-05-11T10:47:41.000+02:00",
          "evidencies": [
            {
              "creation": 1620722995344,
              "responseHeaders": {
                "date": "Tue, 11 May 2021 08:49:56 GMT",
                "_result": "HTTP/1.1 200 OK",
                "server": "Apache",
                "transfer-encoding": "chunked",
                "keep-alive": "timeout=10, max=100",
                "connection": "Keep-Alive",
                "content-type": "multipart/related;start=\"<rootpart*c7ec123e-a155-49e4-8fa2-e4e3e19d7832@example.jaxws.sun.com>\";type=\"application/xop+xml\";boundary=\"uuid:c7ec123e-a155-49e4-8fa2-e4e3e19d7832\";start-info=\"text/xml\"",
                "cache-control": "no-cache"
              },
              "request": "H4sIAAAAAAAA...",
              "response": "H4sIAAAAAAAA...",
              "connectTimedOut": false,
              "readTimedOut": false,
              "elapsedTime": 1419
            }
          ]
        }
      ],
      "consolidacio": {
        "numeroAssentament": "E-00002256-2021",
        "dataAssentament": "01/02/2019 11:48:00.342+00:01",
        "dataPresentacio": "01/02/2019 11:48:00.342+00:01",
        "evidencia": "H4sIAAAAAAAA..."
      }
    }
  }
}
```

Petició
```json
 curl –X GET https://mux-dev.aoc.cat/transaccio/0bad-f45b-b743-f000/consolidacio
 ```
Resposta
```json
 {
  "resultat": "ko",
  "errors": [
    "La transacció no ha estat executada o no ha obtingut cap número d'assentament"
  ]
}
```

#### 4.1.3.3 Eliminació de les dades de consolidació dels assentaments auxiliars

| DELETE https://mux-dev.aoc.cat/transaccio/ **{{idTransaccio}}** /consolidacio/ **{{sentit}}**?assentament= **{{numeroAssentament}}** |
| --- |

Aquesta operació esborra la informació de consolidació prèviament establerta per a un assentament dins de la transacció.

**Petició**

- **idTransaccio** : Identificador de transacció de la qual es vol esborrar les dades de consolidació.

- **sentit** : Tipus d'assentament, entrada o sortida, per al qual es vol esborrar la informació de consolidació.
- **assentament** : Número d'assentament donat pel registre auxiliar. Com a mecanisme de seguretat es demana el número d'assentament auxiliar per a minimitzar la possibilitat de que es pugui esborrar les dades de consolidació d'un assentament per error.

**Resposta**

El missatge de resposta conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot tindre aquest camp son:

**ok** : Si la operació s'ha executat correctament

**ko** : Si s'ha produït algun error en el moment d'executar la operació.

- **errors** : Llista de cadenes de text amb la descripció dels errors que s'hagin pogut produir durant la execució de la operació.

- **idTransaccio** : Identificador de la transacció. El format de l'identificador de la transacció és un número de 64 bits expressat en hexadecimal, en quatre grups: xxxx-xxxx-xxxx-xxxx. Per exemple 0b9f-bf17-e8c1-b031.

- **assentaments** : Bloc _Assentaments_ (veure secció 3.2.5 d'aquest document) amb la informació dels assentaments realitzats a la transacció.

A continuació és mostra alguns exemples d'execució d'aquesta operació:

Petició
```json
curl –X DELETE https://mux-dev.aoc.cat/transaccio/0bad-f45b-b743-f000/consolidacio/E?assentament=E2021000626
```
Resposta
```json 
{
  "resultat": "ok",
  "idTransaccio": "0bad-f45b-b743-f000",
  "tipusTransaccio": "E",
  "ttlRestantEnSegons": 1043,
  "assentaments": {
    "entrada": {
      "auxiliar": [
        {
          "creation": 1620722995145,
          "execucio": 1,
          "resultat": true,
          "codiError": 0,
          "nomReglaEnrutament": "ERES Consorci AOC",
          "nomRegistre": "[DEV] ERES CAOC V2 (Audifilm)",
          "idBustia": "70446C69F6AC",
          "numeroAssentament": "E2021000626",
          "dataAssentament": "2021-05-11T10:49:56.000+02:00",
          "dataPresentacio": "2021-05-11T10:47:41.000+02:00",
          "evidencies": [
            {
              "creation": 1620722995344,
              "responseHeaders": {
                "date": "Tue, 11 May 2021 08:49:56 GMT",
                "_result": "HTTP/1.1 200 OK",
                "server": "Apache",
                "transfer-encoding": "chunked",
                "keep-alive": "timeout=10, max=100",
                "connection": "Keep-Alive",
                "content-type": "multipart/related;start=\"<rootpart*c7ec123e-a155-49e4-8fa2-e4e3e19d7832@example.jaxws.sun.com>\";type=\"application/xop+xml\";boundary=\"uuid:c7ec123e-a155-49e4-8fa2-e4e3e19d7832\";start-info=\"text/xml\"",
                "cache-control": "no-cache"
              },
              "request": "H4sIAAAAAAAA...",
              "response": "H4sIAAAAAAAA...",
              "connectTimedOut": false,
              "readTimedOut": false,
              "elapsedTime": 1419
            }
          ]
        }
      ]
    }
  }
}
```

## 4.2 Entorns <a name="4.2"></a>

Es disposa de dos entorns:

- DESENVOLUPAMENT (proves):

https://mux-dev.aoc.cat/transaccio

https://mux-dev.aoc.cat/bustia

- PREPRODUCCIÓ (entorn per realitzar integracions i proves):

https://mux-pre.aoc.cat/transaccio

https://mux-pre.aoc.cat/bustia

- PRODUCCIÓ:

https://mux.aoc.cat/transaccio

https://mux.aoc.cat/bustia


# 5 Integració de Registres Electrònics <a name="5"></a>

## 5.1 Solució proposada <a name="5.1"></a>

El disseny de MUXv3 intenta pal·liar o reduir els problemes detectats al llarg dels anys de funcionament de la versió actual de MUX (V2). Un d'aquests problemes és la generació d'assentaments 'orfes'. És a dir, aquells assentaments que el registre arriba a generar però no es traslladen a la aplicació que l'ha sol·licitat, pel que aquesta intenta de nou aconseguir un número d'assentament nou, quedant el primer assentament sense ús.

![32](captures/32.png)

Aprofitant el concepte de transacció introduït a MUXv3, els registres electrònics disposen d'un mecanisme que els permet reduir el nombre d'assentaments orfes generats.

Aprofitant l'identificador de la transacció de MUXv3 els registres poden mantenir un registre de l'estat de processament de les transaccions provinents de MUX. Dins d'aquesta taula es pot associar cada _id_ de transacció de MUXv3 i el sentit (entrada / sortida) amb el número d'assentament emès. En el moment de rebre una transacció per al seu processament el registre pot saber ràpidament si la transacció ja havia estat processada, si encara es troba en processament o si és el primer cop que aquesta es tracta.

Dos exemples d'aquest mecanisme s'il·lustren als següents gràfics:

![33](captures/33.png)

__**Gràfic 1**__ - L'assentament es genera correctament i aquest arriba a l'aplicació que el sol·licita però aquesta llança una segona execució de la transacció. El mòdul d'integració del registre amb MUXv3 detecta que ja havia tractat aquesta transacció i directament retorna el número d'assentament generat durant la primera execució de la transacció.

![34](captures/34.png)

__**Gràfic 2**__ - L'assentament es genera correctament però el timeout de xarxa es produeix abans de que l'aplicació l'hagi rebut. Posteriorment l'aplicació torna a executar la transacció per a intentar obtenir el número d'assentament. El mòdul d'integració del registre amb MUXv3 detecta que ja havia tractat aquesta transacció i directament retorna el número d'assentament ja generat durant la primera execució de la transacció.

![35](captures/35.png)

__**Gràfic 3**__ - L'assentament es genera correctament però triga força temps, pel que es produeix un timeout de xarxa i l'assentament no arriba a l'aplicació. Posteriorment l'aplicació executa un segon cop la transacció per a intentar obtenir el número d'assentament però aquest encara no ha estat generat. En aquest cas el mòdul d'integració retorna una resposta d'error indicant que la transacció encara s'està processant. Finalment, l'aplicació executa un tercer cop la transacció. Aquest cop l'assentament ja ha estat generat i l'aplicació n'obté finalment el número.

A mode de proposta, la taula de control de l'estat de processament de les transaccions podria estar formada pels següents elements:

| **Nom del camp** | **Descripció** | **Exemple de valors** |
| --- | --- | --- |
| idTransaccio | Identificador de la transacció MUXv3 | '0fca-1a3c-2e37-6543' |
| sentit | Tipus d'assentament (E=Entrada, S=Sortida) | 'E' |
| estat | Estat de processament | 'pendent', 'error', 'finalitzada' |
| numeroAssentament | Numero d'assentament | 'E/000123-2021 |
| dataAssentament | Data d'assentament | '01/06/2021T13:42:23.000+02:00' |
| dataPresentacio | Data de presentació efectiva | '01/06/2021T13:42:23.000+02:00' |
| codiError | En cas d'error, codi de l'error | 13 |
| missatgeError | En cas d'error, descripció de l'error | 'Manca el camp Identificador de l'interessat' |

La clau primària la formen l'identificador de la transacció i el sentit o tipus d'assentament. Això permet el cas (tot i que estrany) en el qual una transacció interadministrativa de MUXv3 (Sortida + Entrada) genera els dos assentaments contra el mateix registre.

## 5.2 Autenticació de MUXv3 davant el registre electrònic <a name="5.2"></a>

MUXv3 s'autenticarà davant el registre electrònic mitjançant l'ús de _tokens_ JWT. El _token_ vindrà informat a la capçalera HTTP _Authorization_.

Authorization: Bearer \<_token\_jwt_\>

La parametrització del _token_ JWT serà establerta junt amb la unitat de Suport a Integradors en el moment de configurar el nou registre electrònic a MUXv3. S'establirà els camps _issuer_ i _subject_ als valors acordats i el _token_ es signarà amb la clau secreta corresponent.

## 5.3 Missatge de petició <a name="5.3"></a>

La petició que arriba cap al registre electrònic és la transacció de MUXv3 completa, descrita al punt 3 d'aquest document.

A continuació es mostra un exemple de petició HTTP corresponent a una transacció MUXv3 entrant.

```json
POST /registre HTTP/1.1
Host: registre.exemple.cat
Connection: keep-alive
Accept: */*
Accept-encoding: gzip, deflate
User-Agent: Mozilla/4.0 (compatible;
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIiLCJzdWIiOiIiLCJpYXQiOjE2MjMxNDQ1NDMsImV4cCI6MTYyMzE0NDg0MywiYXVkIjpbbnVsbF0sImp0aSI6Imp3dF9ub25jZSJ9.OPZxtziO1kkFkufsknnSunEUBmZFAweFsLVPuWlNucw
Content-type: application/json
Content-length: 19319

{
  "idTransaccio": "0ba6-8133-af41-6006",
  "execucions": 0,
  "tipus": "E",
  "dataCreacio": 1620222901949,
  "control": {
    "aplicacio": "TEST",
    "urlAplicacio": "http://tramits.aoc.cat/tramits?idTramit=1000",
    "observacionsApunt": "Observacions",
    "documentacioFisica": {
      "clau": "3",
      "descripcio": "Sense documentació física"
    },
    "indicadorProva": {
      "clau": "0",
      "descripcio": "Transacció real"
    },
    "usuari": "11111111H",
    "contacteUsuari": "usuari@usuari.com"
  },
  "assumpte": {
    "resum": "Creació d'assentament",
    "observacions": "Observacions",
    "dataPresentacio": 1620222905181,
    "codiAssumpteSegonsDesti": "0A333",
    "referenciaExterna": "Referencia externa",
    "numeroExpedient": "01234567890",
    "numeroExpedientSegonsDesti": "DST0987654321",
    "codiFamilia": "FAM0001",
    "nomFamilia": "Nom familia",
    "codiProcediment": "PROD001",
    "nomProcediment": "Nom procediment",
    "codiSIA": "SIA0001",
    "nomSIA": "Nom SIA",
    "codiTramit": "TRAM0001",
    "nomTramit": "Nom de tràmit"
  },
  "procediment": {
    "destinatari": {
      "clau": "CI",
      "descripcio": "Ciutadà"
    },
    "inici": {
      "clau": "DO",
      "descripcio": "D'ofici"
    },
    "efecteSilenci": {
      "clau": "SN",
      "descripcio": "Segons normativa aplicable"
    },
    "tipusProcediment": {
      "clau": "EC",
      "descripcio": "Extern Comú"
    },
    "rangNormativa": {
      "clau": "RUE",
      "descripcio": "Reglament UE"
    },
    "materia": {
      "clau": "EIN",
      "descripcio": "Energia i Industria"
    },
    "canalAcces": {
      "clau": "ELE",
      "descripcio": "Electrònic"
    },
    "requisitsIdentificacio": {
      "clau": "DNI",
      "descripcio": "DNIe"
    },
    "viaNotificacio": {
      "clau": "CSU",
      "descripcio": "Compareixença a Seu"
    }
  },
  "registres": {
    "origen": {
      "unitatOrganica": {
        "dir3": "A09018933",
        "dir3Pare": "A09018799",
        "ine10": "9821920002",
        "nom": "Consorci Administració Oberta de Catalunya",
        "cif": "Q0801175A",
        "codiTipusVia": "2",
        "tipusVia": "Calle",
        "adressa": "Tánger",
        "numeroVia": "98",
        "codiMunicipi": "080193",
        "nomMunicipi": "Barcelona",
        "codiComarca": "13",
        "nomComarca": "Barcelonès",
        "codiProvincia": "08",
        "nomProvincia": "Barcelona",
        "codiPostal": "08018",
        "codiComunitat": "09",
        "nomComunitat": "Cataluña",
        "codiPais": "724",
        "nomPais": "España",
        "codiAmbitTerritorial": "12",
        "nomAmbitTerritorial": "Central-Autonómico",
        "codiEntitatGeografica": "01",
        "nomEntitatGeografica": "Municipio",
        "codiNivellAdministracio": "2",
        "nomNivellAdministracio": "Administración Autonómica",
        "oficinesAssociades": [
          {
            "dir3": "O00011864",
            "dir3UnitatResponsable": "A09018933",
            "ine10": "9821920002",
            "nom": "Consorcio Administración Abierta de Cataluña (CAOC)",
            "codiTipusVia": "2",
            "tipusVia": "Calle",
            "adressa": "Tanger",
            "numeroVia": "98",
            "codiMunicipi": "080193",
            "nomMunicipi": "Barcelona",
            "codiComarca": "13",
            "nomComarca": "Barcelonès",
            "codiProvincia": "08",
            "nomProvincia": "Barcelona",
            "codiPostal": "08018",
            "codiComunitat": "09",
            "nomComunitat": "Cataluña",
            "codiPais": "724",
            "nomPais": "España",
            "codiEntitatGeografica": "01",
            "nomEntitatGeografica": "Municipio",
            "codiNivellAdministracio": "2",
            "nomNivellAdministracio": "Administración Autonómica",
            "serveisOficina": [
              {
                "codi": "5",
                "descripcio": "Oficina Integrada en SIR"
              },
              {
                "codi": "6",
                "descripcio": "Oficina Integrada en SIR (Envío)"
              },
              {
                "codi": "7",
                "descripcio": "Oficina Integrada en SIR (Recepción)"
              }
            ],
            "unitatsServides": [
              "A09018933"
            ]
          }
        ],
        "unitatPare": {
          "dir3": "A09018799",
          "dir3Pare": "A09002970",
          "ine10": "9610420002",
          "nom": "Departamento de Políticas Digitales y Administraciones Públicas",
          "cif": "S0811001G",
          "codiTipusVia": "37",
          "tipusVia": "Via",
          "adressa": "Laietana",
          "numeroVia": "26",
          "codiMunicipi": "080193",
          "nomMunicipi": "Barcelona",
          "codiComarca": "13",
          "nomComarca": "Barcelonès",
          "codiProvincia": "08",
          "nomProvincia": "Barcelona",
          "codiPostal": "08003",
          "codiComunitat": "09",
          "nomComunitat": "Cataluña",
          "codiPais": "724",
          "nomPais": "España",
          "codiAmbitTerritorial": "12",
          "nomAmbitTerritorial": "Central-Autonómico",
          "codiEntitatGeografica": "01",
          "nomEntitatGeografica": "Municipio",
          "codiNivellAdministracio": "2",
          "nomNivellAdministracio": "Administración Autonómica",
          "codiTipusUnitatOrganica": "CJ",
          "nomTipusUnitatOrganica": "Consejería",
          "oficinesAssociades": [
            {
              "dir3": "O00007257",
              "dir3UnitatResponsable": "A09018799",
              "ine10": "9610420002",
              "nom": "Departamento de Políticas Digitales y Administraciones Públicas",
              "codiTipusVia": "37",
              "tipusVia": "Via",
              "adressa": "Laietana",
              "numeroVia": "26",
              "codiMunicipi": "080193",
              "nomMunicipi": "Barcelona",
              "codiComarca": "13",
              "nomComarca": "Barcelonès",
              "codiProvincia": "08",
              "nomProvincia": "Barcelona",
              "codiPostal": "08003",
              "codiComunitat": "09",
              "nomComunitat": "Cataluña",
              "codiPais": "724",
              "nomPais": "España",
              "codiEntitatGeografica": "01",
              "nomEntitatGeografica": "Municipio",
              "codiNivellAdministracio": "2",
              "nomNivellAdministracio": "Administración Autonómica",
              "tipusOficina": "1",
              "dadesContacte": {
                "telefon": [
                  "935671700"
                ],
                "fax": [
                  "935671769"
                ]
              },
              "serveisOficina": [
                {
                  "codi": "1",
                  "descripcio": "Oficina de Registro"
                },
                {
                  "codi": "5",
                  "descripcio": "Oficina Integrada en SIR"
                },
                {
                  "codi": "7",
                  "descripcio": "Oficina Integrada en SIR (Recepción)"
                }
              ],
              "unitatsServides": [
                "A09018799"
              ]
            }
          ],
          "unitatPare": {
            "dir3": "A09002970",
            "dir3Pare": "A99999999",
            "ine10": "0000000001",
            "nom": "Generalitat de Catalunya",
            "cif": "S0811001G",
            "codiMunicipi": "080193",
            "codiComarca": "13",
            "nomComarca": "Barcelonès",
            "codiProvincia": "08",
            "codiPostal": "08003",
            "codiAmbitTerritorial": "12",
            "nomAmbitTerritorial": "Central-Autonómico",
            "codiNivellAdministracio": "2",
            "nomNivellAdministracio": "Administración Autonómica",
            "codiTipusEntitatPublica": "CA",
            "nomTipusEntitatPublica": "Comunidad Autonoma",
            "codiTipusUnitatOrganica": "PR",
            "nomTipusUnitatOrganica": "Presidencia",
            "dadesContacte": {
              "telefon": [
                "934024600"
              ],
              "fax": [
                "933183488"
              ],
              "URL": [
                "http://www.president.cat/pres_gov/president/ca/president-mas/contactar.html"
              ]
            },
            "unitatPare": {
              "dir3": "A99999999",
              "dir3Pare": "A99999999",
              "nom": "Comunidades Autonomas",
              "codiAmbitTerritorial": "NA",
              "nomAmbitTerritorial": "N/A",
              "codiNivellAdministracio": "2",
              "nomNivellAdministracio": "Administración Autonómica"
            }
          }
        }
      },
      "sarcat": {
        "urPK": "9040",
        "idCentreDestinacioInterna": 79507
      }
    },
    "desti": {
      "unitatOrganica": {
        "dir3": "A09018933",
        "dir3Pare": "A09018799",
        "ine10": "9821920002",
        "nom": "Consorci Administració Oberta de Catalunya",
        "cif": "Q0801175A",
        "codiTipusVia": "2",
        "tipusVia": "Calle",
        "adressa": "Tánger",
        "numeroVia": "98",
        "codiMunicipi": "080193",
        "nomMunicipi": "Barcelona",
        "codiComarca": "13",
        "nomComarca": "Barcelonès",
        "codiProvincia": "08",
        "nomProvincia": "Barcelona",
        "codiPostal": "08018",
        "codiComunitat": "09",
        "nomComunitat": "Cataluña",
        "codiPais": "724",
        "nomPais": "España",
        "codiAmbitTerritorial": "12",
        "nomAmbitTerritorial": "Central-Autonómico",
        "codiEntitatGeografica": "01",
        "nomEntitatGeografica": "Municipio",
        "codiNivellAdministracio": "2",
        "nomNivellAdministracio": "Administración Autonómica",
        "oficinesAssociades": [
          {
            "dir3": "O00011864",
            "dir3UnitatResponsable": "A09018933",
            "ine10": "9821920002",
            "nom": "Consorcio Administración Abierta de Cataluña (CAOC)",
            "codiTipusVia": "2",
            "tipusVia": "Calle",
            "adressa": "Tanger",
            "numeroVia": "98",
            "codiMunicipi": "080193",
            "nomMunicipi": "Barcelona",
            "codiComarca": "13",
            "nomComarca": "Barcelonès",
            "codiProvincia": "08",
            "nomProvincia": "Barcelona",
            "codiPostal": "08018",
            "codiComunitat": "09",
            "nomComunitat": "Cataluña",
            "codiPais": "724",
            "nomPais": "España",
            "codiEntitatGeografica": "01",
            "nomEntitatGeografica": "Municipio",
            "codiNivellAdministracio": "2",
            "nomNivellAdministracio": "Administración Autonómica",
            "serveisOficina": [
              {
                "codi": "5",
                "descripcio": "Oficina Integrada en SIR"
              },
              {
                "codi": "6",
                "descripcio": "Oficina Integrada en SIR (Envío)"
              },
              {
                "codi": "7",
                "descripcio": "Oficina Integrada en SIR (Recepción)"
              }
            ],
            "unitatsServides": [
              "A09018933"
            ]
          }
        ],
        "unitatPare": {
          "dir3": "A09018799",
          "dir3Pare": "A09002970",
          "ine10": "9610420002",
          "nom": "Departamento de Políticas Digitales y Administraciones Públicas",
          "cif": "S0811001G",
          "codiTipusVia": "37",
          "tipusVia": "Via",
          "adressa": "Laietana",
          "numeroVia": "26",
          "codiMunicipi": "080193",
          "nomMunicipi": "Barcelona",
          "codiComarca": "13",
          "nomComarca": "Barcelonès",
          "codiProvincia": "08",
          "nomProvincia": "Barcelona",
          "codiPostal": "08003",
          "codiComunitat": "09",
          "nomComunitat": "Cataluña",
          "codiPais": "724",
          "nomPais": "España",
          "codiAmbitTerritorial": "12",
          "nomAmbitTerritorial": "Central-Autonómico",
          "codiEntitatGeografica": "01",
          "nomEntitatGeografica": "Municipio",
          "codiNivellAdministracio": "2",
          "nomNivellAdministracio": "Administración Autonómica",
          "codiTipusUnitatOrganica": "CJ",
          "nomTipusUnitatOrganica": "Consejería",
          "oficinesAssociades": [
            {
              "dir3": "O00007257",
              "dir3UnitatResponsable": "A09018799",
              "ine10": "9610420002",
              "nom": "Departamento de Políticas Digitales y Administraciones Públicas",
              "codiTipusVia": "37",
              "tipusVia": "Via",
              "adressa": "Laietana",
              "numeroVia": "26",
              "codiMunicipi": "080193",
              "nomMunicipi": "Barcelona",
              "codiComarca": "13",
              "nomComarca": "Barcelonès",
              "codiProvincia": "08",
              "nomProvincia": "Barcelona",
              "codiPostal": "08003",
              "codiComunitat": "09",
              "nomComunitat": "Cataluña",
              "codiPais": "724",
              "nomPais": "España",
              "codiEntitatGeografica": "01",
              "nomEntitatGeografica": "Municipio",
              "codiNivellAdministracio": "2",
              "nomNivellAdministracio": "Administración Autonómica",
              "tipusOficina": "1",
              "dadesContacte": {
                "telefon": [
                  "935671700"
                ],
                "fax": [
                  "935671769"
                ]
              },
              "serveisOficina": [
                {
                  "codi": "1",
                  "descripcio": "Oficina de Registro"
                },
                {
                  "codi": "5",
                  "descripcio": "Oficina Integrada en SIR"
                },
                {
                  "codi": "7",
                  "descripcio": "Oficina Integrada en SIR (Recepción)"
                }
              ],
              "unitatsServides": [
                "A09018799"
              ]
            }
          ],
          "unitatPare": {
            "dir3": "A09002970",
            "dir3Pare": "A99999999",
            "ine10": "0000000001",
            "nom": "Generalitat de Catalunya",
            "cif": "S0811001G",
            "codiMunicipi": "080193",
            "codiComarca": "13",
            "nomComarca": "Barcelonès",
            "codiProvincia": "08",
            "codiPostal": "08003",
            "codiAmbitTerritorial": "12",
            "nomAmbitTerritorial": "Central-Autonómico",
            "codiNivellAdministracio": "2",
            "nomNivellAdministracio": "Administración Autonómica",
            "codiTipusEntitatPublica": "CA",
            "nomTipusEntitatPublica": "Comunidad Autonoma",
            "codiTipusUnitatOrganica": "PR",
            "nomTipusUnitatOrganica": "Presidencia",
            "dadesContacte": {
              "telefon": [
                "934024600"
              ],
              "fax": [
                "933183488"
              ],
              "URL": [
                "http://www.president.cat/pres_gov/president/ca/president-mas/contactar.html"
              ]
            },
            "unitatPare": {
              "dir3": "A99999999",
              "dir3Pare": "A99999999",
              "nom": "Comunidades Autonomas",
              "codiAmbitTerritorial": "NA",
              "nomAmbitTerritorial": "N/A",
              "codiNivellAdministracio": "2",
              "nomNivellAdministracio": "Administración Autonómica"
            }
          }
        }
      },
      "sarcat": {
        "urPK": "9040",
        "idCentreDestinacioInterna": 79507
      }
    }
  },
  "interessats": [
    {
      "tipusIdentificador": {
        "clau": "NIF",
        "muxv2": "NIF",
        "sarcatEntrada": "1",
        "sarcatSortida": "1",
        "eres": "PF",
        "eacat": "07",
        "descripcio": "Número d'Identificació Fiscal"
      },
      "identificador": "11111111H",
      "informacioIdentificador": {
        "valid": true,
        "tipusIdentificador": "NIF",
        "tipusPersona": "F",
        "tipusEntitat": "Persona física",
        "identificador": "11111111H"
      },
      "raoSocial": "Raó Social S.L.",
      "nomSentit": "Quico",
      "nom": "Francesc",
      "cognom1": "Cognom",
      "cognom2": "Cognom",
      "genere": {
        "clau": "H",
        "descripcio": "Home"
      },
      "municipi": "Barcelona",
      "provincia": "Barcelona",
      "pais": "Espanya",
      "adressaCompleta": "Carrer Mallorca 23 1er 1a",
      "tipusVia": {
        "clau": "4",
        "descripcio": "Carrer"
      },
      "nomVia": "MALLORCA",
      "numeroInferior": "23",
      "pis": "1er",
      "porta": "1a",
      "codiPostal": "08028",
      "email": "email@mail.com",
      "adressaElectronicaHabilitada": "email@mail.com",
      "canalPreferentComunicacio": "mail",
      "observacions": "Observacions",
      "representant": {
        "tipusIdentificador": {
          "clau": "VAT",
          "muxv2": "VATNumber",
          "sarcatEntrada": "2",
          "sarcatSortida": "2",
          "eres": "PJ",
          "eacat": "01",
          "descripcio": "Identificador Fiscal Europeu"
        },
        "identificador": "FR56384423477",
        "informacioIdentificador": {
          "valid": true,
          "tipusIdentificador": "VAT",
          "tipusPersona": "J",
          "tipusEntitat": "Identificador fiscal europeu [França]",
          "identificador": "FR56384423477"
        },
        "raoSocial": "Societat S.L.",
        "nom": "Nom",
        "cognom1": "Cognom",
        "cognom2": "Cognom",
        "municipi": "Ripollet",
        "provincia": "Barcelona",
        "pais": "Espanya",
        "tipusVia": {
          "clau": "19",
          "descripcio": "Avinguda"
        },
        "nomVia": "Meridiana",
        "numeroSuperior": "25",
        "numeroInferior": "23",
        "bloc": "B",
        "escala": "2",
        "pis": "3",
        "porta": "F",
        "codiPostal": "08291",
        "email": "email@email.cat",
        "adressaElectronicaHabilitada": "email@email.cat",
        "canalPreferentComunicacio": "mail",
        "observacions": "Observacions"
      }
    }
  ],
  "instanciaGenerica": {
    "exposa": "Exposo que...",
    "solicita": "Sol·licito que..."
  },
  "dadesLliures": {
    "Factura": {
      "total": 23778,25,
      "numFactura": "2021-00000001"
    },
    "anyFactura": 2021
  },
  "ttl": 20
}
```


## 5.4 Missatge de resposta <a name="5.4"></a>

El missatge de resposta que el registre ha de retornar conté els següents camps:

- **resultat** : Indica si la operació s'ha realitzat correctament o no. Els valors que pot adoptar aquest camp son:

**true** : Si la transacció s'ha processat correctament i s'ha generat un número d'assentament.

**false** : Si s'ha produït algun error en el moment al processar la transacció.

- **codiError** : Codi numèric de l'error produït al processar la transacció.

- **missatgeError** : Descripció de l'error produït al processar la transacció.

- **idTransaccio** : Identificador de la transacció processada.

- **tipus** : Tipus d'assentament generat:

**E** : Assentament d'entrada.

**S** : Assentament de sortida.

- **numeroAssentament** : Número d'assentament emès.

- **dataAssentament** : Data d'assentament. Donat que MUXv3 intentarà processar la data el format d'aquesta ha de seguir el següent format:</b>dd/MM/YYYY'T'HH:mm:ss.SSSZZZZZ

per exemple:</b>16/06/2021T18:39:14.729+02:00

- **dataPresentacio** : Data de presentació. Donat que MUXv3 intentarà processar la data el format d'aquesta ha de seguir el següent format:</b>dd/MM/YYYY'T'HH:mm:ss.SSSZZZZZ

per exemple:</b>01/12/2020T12:20:03.023+01:00

A continuació es mostra un exemple de resposta del registre electrònic:

```json
{
   "resultat":true,
   "idTransaccio":"0b8b-b8d2-4941-e02b",
   "tipus":"E",
   "numeroAssentament":"E/0000000001/2021",
   "dataAssentament":"16/06/2021T13:43:48.857+02:00",
   "dataPresentacio":"16/03/2021T16:09:38.000+01:00"
}
 ```

## 5.5 Mòdul d'exemple <a name="5.5"></a>

Amb la finalitat de facilitar la realització de proves i poder familiaritzar-se amb la solució proposada per a la construcció d'un connector de registre electrònic de MUXv3, s'ha desenvolupat una petita peça de software amb una implementació molt bàsica d'aquest model.

Concretament, es tracta d'un projecte per al IDE _Eclipse_ que genera un fitxer _.war_ que es pot desplegar, per exemple, a un contenidor de _servlets_ tipus _Tomcat_.

![36](captures/36.png)

Aquest mòdul està dirigit a ser un exemple, una implementació de referència, del model d'integració amb MUXv3 per als registres electrònics. Ara bé, aquesta implementació no està preparada per a tindre en compte altres factors a considerar en un entorn productiu com la concurrència o la persistència de les dades tractades.