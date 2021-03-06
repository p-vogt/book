# Hyperledger Fabric Extension
Autor: Cem Basoglu

*"Hyperledger ist eine bereichsübergreifende Open-Source-Initiative zur Förderung
branchenübergreifender Blockchain-Technologien. An diesem weltweiten
Kooperationsprojekt, das von The Linux Foundation gehostet wird, nehmen führende
Unternehmen aus Branchen wie Banken und Finanzen, Internet of Things,
Supply-Chain, Fertigung und IT teil"* [[HYPE18](#ref_hype18)].

Zu den von der Hyperledger Initiative geförderten Projekten gehört unter anderem
auch das von IBM initiierte Fabric Framework. Das Fabric Framework besteht aus
einer Distributed Ledger Plattform und diversen Client Bibliotheken zum Zugriff
auf den Ledger. Weitere Informationen zum Fabric Framework sind im Kapitel
[Populäre Blockchain Plattformen](/blockchain/plattformen/#hyperledger)
verfügbar.

Um aus einer Webapplikation heraus mit dem Distributed Ledger zu kommunizieren,
erfordert das derzeitige [Fabric SDK](https://fabric-sdk-node.github.io/)
zwangsläufig eine serverseitige Schnittstelle zwischen Hyperledger Fabric und
der clientseitigen Webapplikation. Diese Restriktion ist darin begründet, dass
der Fabric Client zum einen das `fs` und `path`-Modul aus Node.js benötigt und
zum anderen eine gRPC-Implementierung verwendet, die nicht im Browser lauffähig
ist [[SANG18](#ref_sang18)].

Für die Authentifizierung eines Nutzers gegenüber dem Distributed Ledger,
verwendet das Hyperledger Fabric Framwork x509-Zertfikate. Der mit dem
Zertifikat verbundene private Schlüssel hingegen wird für die Signierung der
Anfragen verwendet. Da das Fabric SDK die Aufgabe der Authentifizierung
gegenüber dem Distributed Ledger und die Signierung der Anfragen übernimmt,
müssen die Zertifikate und die privaten Schlüssel, je nach Implementierung der
serverseitigen Schnittstelle, auf dem Server vorgehalten oder an diesen mit
jeder Anfrage übertragen werden. Beide Varianten der Implementierung stellen
jedoch ein großes Sicherheitsrisko dar, da der private Schlüssel nur
clientseitig vorgehalten werden sollte [[MINY18](#ref_miny18)].

Moderne Browser wie Chrome, Firefox, Opera oder auch Microsoft Edge, erlauben es
Entwicklern eigene Erweiterungen zu implementieren. Durch die Implementierung
einer Erweiterung können die oben genannten Sicherheitsrisiken umgangen werden,
indem die Erweiterung die Verwahrung der Zertifikate und die Signierung der
Anfragen übernimmt. Diese Erweiterung kann anschließend aus der clientseitigen
Webapplikation genutzt werden, um mit dem Distributed Ledger zu interagieren.
Somit verbleiben privaten Schlüssel jederzeit beim Client und nur die mit
privaten Schlüssel signierten Anfragen werden über das Netzwerk übertragen.

## Anforderungen
Die sichere Verwahrung der Zertifikate und privaten Schlüssel ist die
zentrale Anforderung an die Fabric Chrome Erweiterung. Dazu müssen die
Zertifikate und privaten Schlüssel vor einem unberechtigten Zugriff auf
Dateisystemebene und auch aus einer Webapplikation heraus geschützt werden.

Des Weiteren muss der Benutzer die Möglichkeit haben, neue Zertifikate zu hinterlegen und
bestehende Zertifikate zu verwalten. Dies soll direkt über die Erweiterung im
Browser möglich sein und muss zusätzlich mit einem Passwort gesichert werden.

Für die Interaktion mit der Erweiterung muss es eine Javascript-Schnittstelle
bereitgestellt werden, die in die clientseitige Webapplikation eingebunden werden
kann. Diese soll berechtigten Webapplikationen die Möglichkeit geben zu prüfen, ob
ein Zertifikat für die aktuelle Webapplikation vorhanden ist. Zusätzlich müssen
allgemeine Informationen über das Zertifikat abgerufen werden können. Für die
Interaktion mit dem Distributed Ledger müssen beliebige Chaincode Funktionen
aufgerufen werden können.

## Analyse
Um die Anforderungen umzusetzen, muss zunächst untersucht werden welche
Möglichkeiten bestehen, um aus einer Chrome Extension mit einem gRPC-Dienst zu
interagieren und wie aus einer clientseitigen Webapplikation mit der
Extension kommuniziert werden kann.

### Fabric Client
Wie bereits einleitend erwähnt, benötigt das [Fabric SDK](https://fabric-sdk-node.github.io/)
die Module `fs` und `path` aus Node.js, um Zertifikate für die Wiederverwendung
abzulegen. Diese Module sind jedoch nicht in einer Browser bzw. Extension
Umgebung verfügbar, sodass für die Persistierung eine alternative Lösung
verwendet werden muss. Zusätzlich verwendet das SDK die [gRPC-Node](https://www.npmjs.com/package/grpc)
Bibliothek um mit dem Fabric Netzwerk zu kommunizieren. Dieser Client ist jedoch
als natives C++ Module für Node.js implementiert, was die Verwendbarkeit im
Browser weiter einschränkt.

Für die Implementierung einer Chrome-Extension müssen daher
die Funktionalitäten der Fabric SDK neu implementiert werden. Zu diesen
Funktionalitäten zählt die Interaktion mit der gRPC-Schnittstelle des Fabric
Netzwerks und das Erzeugen der kryptografischen Signatur zur Signierung der
Anfragen. Um auch aus der Chrome-Extension auf das Fabric Netzwerk zugreifen zu
können, muss auf eine gRPC Bibliothek zurückgegriffen werden, die auch aus dem
Browser heraus genutzt werden kann. Zu diesem Zweck sollen an dieser Stelle
alternative gRPC Bibliotheken, auf die Verwendbarkeiten im Browser betrachtet
werden.

#### gRPC-js
Die gRPC-js Bibliothek ist ein, ausschließlich in Javascript implementierter,
experimenteller gRPC-Client, wodurch die Abhängigkeit zu der C++ Implementierung
wegfällt [[GRJS18](#ref_grjs18)]. Diese Bibliothek verwendet das
[HTTP/2](https://nodejs.org/api/http2.html)-Modul aus Node.js um mit dem
gRPC-Server zu kommunizieren und unterstützt somit ebenfalls Dienste mit
bidirektionalen Datenströmen zwischen Client und Server.

Das HTTP/2-Protokoll wird zwar ebenfalls von modernen Browsern unterstützt,
diese stellen das Protokoll aber nur implizit über die `fetch`- bzw.
`XMLHttpRequest`-Schnittstelle bereit. Daher kann die gRPC-js Bibliothek nicht im Browser
eingesetzt werden, da die Schnittstelle des HTTP/2-Moduls in Node.js nicht kompatibel ist,
mit der Schnittstelle im Browser.

#### gRPC-Web
Für den Zugriff auf einen gRPC-Dienst aus dem Browser gibt es seit Anfang 2018
eine öffentliche Betaphase der offiziellen gRPC-Web-Bibliothek
[[GRWE18](#ref_grwe18)]. Diese nutzt, je nach Browser und Verfügbarkeit,
die `fetch`- bzw. `XMLHttpRequest`-Schnittstelle um gRPC-Dienste über HTTP/2-Requests
anzusprechen.

Aufgrund einiger Einschränkungen im Browser, wie z.B. die fehlende Unterstützung
für HTTP/2-Trailers oder der fehlende Zugriff auf das HTTP/2 framing, verwendet
die gRCP-Web-Bibliothek eine vom [nativen gRPC Protokoll](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
abweichende Implementierung [[GRWP18](#ref_grwp18)]. Das gRPC-Web-Protokoll wird
jedoch mit zunehmender Browserunterstützung der [Streams Spezifikation](https://streams.spec.whatwg.org/)
obsolet werden, sodass auch im Browser das native gRPC Protokoll implementiert
werden kann.

Daher ist für die Nutzung dieser Bibliothek zwangsläufig ein Proxy notwendig,
der die Anfragen von dem gRPC-Web-Protokoll in das native gRPC-Protokoll umformt
und an den gRPC-Dienst weiterleitet. Dazu stellt die Bibliothek ein nginx
Module bereit, die vor der Verwendung jedoch zunächst noch kompiliert werden
muss [[NGIN18](#ref_ngin18)].

#### gRPC-Web-Client
Ähnlich wie die gRPC-Web-Bibliothek, ist auch diese inoffizielle Bibliothek
explizit für die Verwendung im Browser konzipiert [[INWE18](#ref_inwe18)].
Dazu implementiert es jedoch ebenfalls ein vom nativen gRPC-Protokoll
abweichendes Protokoll, um die Limitierung im Browser zu umgehen.

Im Gegensatz zu der gRPC-Web-Bibliothek, bietet diese Bibliothek jedoch die
Möglichkeit, aus den Protcol Buffers einen entsprechenden gRPC-Server zu
generieren, mit dem direkt aus dem Browser kommuniziert werden kann.

Für eine *fremde* gRPC-Schnittstelle, die das native gRPC-Protkoll implementieren, ist
jedoch weiterhin ein Proxy für die Konvertierung zwischen Protokollen notwendig.
Dazu stellt die Bibliothek ein generisches golang-Tool bereit, der über
Startparameter konfiguriert werden kann.

Ein weiteres nützliches Feature ist die Möglichkeit aus den Protocol Buffers
entsprechende DTO und Service Klassen für Typescript zu generieren.

### Chrome Extension
Für den Entwurf einer geeigneten Architektur und Wahl der zusätzlich benötigten
Bibliotheken muss der grundsätzliche Aufbau einer Chrome Extensions betrachtet
werden. Die Kernfunktionen der Erweiterung sind die verschlüsselte Persistierung
der Zertifikate inkl. der privaten Schlüssel und die Kommunikation mit der
Webapplikation.

Dazu wird zunächst untersucht, wie die Kommunikation zwischen der im Browser
ausgeführten Webapplikation und er Erweiterung realisiert werden
kann. Anschließend werden die Möglichkeiten zur Persistierung der Zertifikate
und privaten Schlüssel betrachtet.

#### Kommunikation
Die Kommunikation zwischen den in [Abbildung 2.2.2.2](#img_2222) abgebildeten
Komponenten einer Chrome Extension, kann mittels der Message-Schnittstelle realisiert
werden [[CHMA18](#ref_chma18)].

![Chrome Extension Messaging](./messagingarc.png)

<a name="img_2222">Abbildung 2.2.2.2</a> - Chrome Extension Architektur *Quelle:* [[CHEX18](#ref_chex18)]

##### Background Script
Das Background Script wird als Singleton initial beim Browserstart geladen und
ist während der gesamten Laufzeit des Browsers verfügbar. Es ist der zentrale
Event-Handler der Erweiterung und kann ebenfalls für Seitenübergreifende State
Persistierung verwendet werden.

##### Popup
Diese Komponente einer Chrome Erweiterung enthält dass als Popup angezeigte
User Interface und die dazugehörige Applikationslogik. Die Anzeige des Popups
kann nur vom Benutzer durch den Klick auf die Schaltfläche initiiert werden,
wodurch die Komponente erst erzeugt und initialisiert wird. Beim Schließen des
Popup werden auch die Komponenten wieder zerstört, sodass State-Informationen
anderweitig persistiert werden müssen.

##### Content Script
Für jeden offenen Tab im Browser wird ein sogenanntes Content Script geladen,
das in Kontext der aktuellen Webapplikation läuft. Obwohl das Content Script und
die Webapplikation einen Kontext teilen, sind die Laufzeitumgebungen strikt
voneinander isoliert. Ein Content Script kann daher zwar auf den DOM der
aktuellen Seite zugreifen und diesen manipulieren, aber keine Javascript-Objekte
oder Funktion der Seite benutzen. Im Umkehrschluss kann daher auch die Seite
keine Funktion aus dem Content Script verwenden.

Damit eine berichtigte Webapplikation dennoch auf die Zertifikate zugreifen und
mittels der Extension Daten vom Distributed Ledger abrufen kann, kann die
`window.postMessage` für das Marshalling und das Dispatchen der Funktionsaufrufe
verwendet werden [[CHPC18](#ref_chpc18)]. Um diesen Vorgang zu vereinfachen, kann
ebenfalls die [post-message-stream](https://github.com/kumavis/post-message-stream)-Bibliothek
verwendet werden.


#### Persistierung
Ähnlich wie in der Browser Umgebung, kann auch in einer Chrome Extension die
LocalStorage-Schnittstelle verwendet werden, um die Zertifikate und privaten Schlüssel zu
persistieren. Für die Verschlüsselung der Daten kann die
[CryptoJS](https://www.npmjs.com/package/crypto-js)-Bibliothek genutzt werden.

## Entwurf
Für die Verschlüsselung der im Browser Storage abgelegten Zertifikate wird das
vom Benutzer eingegebene Passwort verwendet, welches gleichzeitig die Fabric
Extension vor fremden Zugriff schützt. Für den Zugriff aus einer berechtigten
Webapplikation muss die Fabric Extension einmalig entsperrt werden.
Anschließend kann aus der Webapplikation, mittels einer Javascript Bibliothek,
auf die Zertifikate zugegriffen werden. In den Attributen der Zertifikate ist
eine URL in Form eines regulären Ausdrucks hinterlegt der eine
Webapplikation autorisiert auf das Zertifikat zuzugreifen.

Wie bereits in der [Analyse](#fabric-client) erläutert, ist für die
Kommunikation mit einem gRPC-Dienst zwangsläufig eine serverseitige
Schnittstelle zwischen der clientseitigen Webapplikation und dem
Hyperledger Fabric Netzwerk notwendig. Da aus den vorgestellten Bibliotheken
nur [gRPC-Web](#grpc-web) und [gRPC-Web-Client](#grpc-web-client) tatsächlich
für die Verwendung im Browser konzipiert wurden, wird aus folgenden Gründen die
gRPC-Web-Client-Bibliothek für die Implementierung der Fabric Extension
eingesetzt.

* Typescript Code Generator für Protocol Buffers
* golang-Tool für Reverse-Proxy verfügbar
* NPM Paket verfügbar mit 4077 wöchentliche NPM Installationen
* Erste Version seit März 2017 öffentlich verfügbar

Mit der gRPC-Web-Client-Bibliothek ist zwar nach wie vor eine serverseitige
Schnittstelle notwendig, jedoch können nun die Authentifizierung und die
Signierung der Anfragen, in den Browser verlagert werden. Damit stellt die
serverseitige Schnittstelle lediglich einen Reverse-Proxy dar, wodurch der
private Schlüssel zu keinem Zeitpunkt über Netzwerk übertragen oder auf dem
Server vorgehalten werden muss.

### Architektur

![Fabric Extension Architektur](./architecture.png)

<a name="img_2231">Abbildung 2.2.3.1</a> - Fabric Extension Architektur

#### Fabric Client (reimplemented)
Diese Komponente ersetzt die offiziele Fabric Node SDK Bibliothek und
implementiert daher alle für die Signierung und Authentifizierung
notwendigen Funktionalitäten. Zusätzlich nutzt es die gRPC-Web-Client-Bibliothek
für die Interaktion mit dem Hyperledger Fabric Framework.

#### Background Script
Für die Fabric Extension übernimmt das Background Script die Persistierung der
Zertifikate und privaten Schlüssel und stellt diese, nach erstmaliger Eingabe
des Passworts im Popup, über die Messaging-Schnittstelle den restlichen
Komponenten bereit.

#### Popup / Certificate Service
Die Benutzerschnittstelle wird unter der Nutzung des Angular Frameworks in
dieser Komponente implementiert. Über diese kann der Benutzer, nach der Eingabe
des Passworts, die Zertifikate einsehen und verwalten. Über das Certificate
Service werden dabei die Zertifikate vom Background Script abgerufen bzw. neue
Zertifikate persistiert.

#### Fabric Extension Client
Die Fabric Extension Client stellt die Funktionalitäten zum Zugriff auf die
Zertifikate sowie die Interaktion mit dem Distributed Ledger bereit. Dazu
kapselt es die Logik, um mit dem Content Script zu kommunizieren und bietet der
clientseitigen Webapplikation, eine Javascript-Schnittstelle zur Interaktion mit der
Erweiterung an.

#### Content Script
Die Komponente hält den State der aktuellen Webapplikation und dient als
Schnittstelle zwischen dem Fabric Client und dem Fabric Extension Client. Dazu
wird die post-message-stream-Bibliothek verwendet, um die Funktionsaufrufe über
das Fenster-Objekt aus der Webapplikation an das Content Script weiterzuleiten
und Ergebnisse zurückzuliefern.

## Evaluation
Zur Evaluation der Fabric Extension wird eine Webapplikation implementiert,
die Information aus der digitalen Patientenakte vom [Health Ledger Projekt](https://github.com/SGSE18/health-ledger/)
anzeigt. Dabei steht jedoch die Demonstration der Funktionalität der Erweiterung
im Vordergrund.

## Referenzen

<a name="ref_chex18">[CHEX18]</a> Google Chrome Extension Dokumentation - Content Scripts [Online](https://developer.chrome.com/extensions/overview#contentScripts)

<a name="ref_chma18">[CHMA18]</a> Google Chrome Extension Dokumentation - Message Passing [Online](https://developer.chrome.com/extensions/messaging)

<a name="ref_chpc18">[CHPC18]</a> Google Chrome Extension Dokumentation - Host Page Communication [Online](https://developer.chrome.com/extensions/content_scripts#host-page-communication)

<a name="ref_grwe18">[GRWE18]</a> Github: gRPC for Web Clients [Online](https://github.com/grpc/grpc-web)

<a name="ref_grjs18">[GRJS18]</a> Github: Pure JavaScript gRPC Client [Online](https://github.com/grpc/grpc-node/tree/master/packages/grpc-js-core)

<a name="ref_grwp18">[GRWP18]</a> Github: gRPC Web Protocol [Online](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md)

<a name="ref_hype18">[HYPE18]</a> IBM Blockchain auf Basis von Hyperledger Fabric der Linux Foundation [Online](https://www.ibm.com/blockchain/de-de/hyperledger.html)

<a name="ref_inwe18">[INWE18]</a> Github: gRPC Web implementation for Golang and TypeScript [Online](https://github.com/improbable-eng/grpc-web)

<a name="ref_miny18">[MINY18]</a> Min Yu: 2018 Projects - Project 5: Hyperledger Fabric Chrome Extension [Online](https://wiki.hyperledger.org/internship/project_ideas)

<a name="ref_ngin18">[NGIN18]</a> Github: nginx Gateway [Online](https://github.com/grpc/grpc-web/tree/master/net/grpc/gateway/nginx)

<a name="ref_sang18">[SANG18]</a> Siddesh Sangodkar: Browser compatible fabric-node-sdk [Online](https://jira.hyperledger.org/browse/FAB-8129)
