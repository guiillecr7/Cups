# AA2 – CUPS: Servidor d'impressió amb impressora virtual

## 0. Escenari de partida

S'utilitza el mateix escenari que a la PoC de NFS:

- **Servidor (SERVER):** Ubuntu Server amb dues interfícies de xarxa: una en mode **NAT** (sortida a Internet i actualitzacions) i una en mode **Host-Only** (comunicació amb el client). L'adreça del servidor a la xarxa Host-Only és `192.168.10.1`.
- **Client:** Zorin OS Desktop, connectat a la mateixa xarxa Host-Only.

---

## 1. Instal·lació de CUPS

Primer s'actualitza la llista de paquets del servidor i després s'instal·la CUPS.

```bash
sudo apt update
sudo apt install cups
```

`apt update` descarrega els índexs dels repositoris. Al final indica que hi ha 27 paquets actualitzables, però no cal actualitzar-los per a aquesta pràctica.

![apt update](images/01-apt-update.png)

La captura següent mostra l'historial de comandes (`history`), on es veu la comanda `sudo apt install cups` executada al servidor.

![apt install cups](images/02-apt-install-cups.png)

---

## 2. Instal·lació de la impressora virtual cups-pdf

Per simular una impressora de xarxa sense maquinari físic s'instal·la el paquet `printer-driver-cups-pdf`, que afegeix a CUPS un backend que, en lloc d'imprimir en paper, genera un fitxer PDF al servidor.

```bash
sudo apt install printer-driver-cups-pdf
```

Es va instal·lar un sol paquet nou (`printer-driver-cups-pdf 3.0.2`, uns 25,9 kB de descàrrega). El procés també va actualitzar els fitxers PPD de CUPS (`PPD for printer PDF updated`).

A la sortida apareix un avís de `needrestart` sobre un nou kernel pendent i serveis que es reiniciarien. No afecta a la pràctica i només indica que convé reiniciar el sistema en algun moment.

![Instal·lació de cups-pdf](images/03-install-cups-pdf.png)

---

## 3. Configuració de l'administració de CUPS

Per defecte, CUPS només escolta a la màquina local i no permet administració remota. Per poder administrar-lo des del navegador del client i compartir la impressora per la xarxa s'edita el fitxer de configuració:

```bash
sudo nano /etc/cups/cupsd.conf
```

Els paràmetres rellevants del fitxer configurat són:

| Directiva | Valor | Funció |
|---|---|---|
| `Port 631` | | Fa que CUPS escolti al port 631 a totes les interfícies de xarxa (i no només a `localhost`), de manera que és accessible a `192.168.10.1:631`. |
| `Listen /run/cups/cups.sock` | | Manté el socket local per a les connexions de la mateixa màquina. |
| `Browsing yes` | | Anuncia les impressores compartides a la xarxa local. |
| `BrowseLocalProtocols dnssd` | | Protocol utilitzat per anunciar les impressores (DNS-SD/Bonjour). |
| `WebInterface Yes` | | Habilita la interfície web d'administració. |
| `<Location />` | `Order allow,deny` / `Allow all` | Permet l'accés al servidor des de qualsevol adreça de la xarxa. |
| `<Location /admin>` | `AuthType Default` / `Require user @SYSTEM` / `Allow all` | Permet accedir a les pàgines d'administració des de la xarxa, però exigeix autenticar-se amb un usuari administrador del sistema. |

![Fitxer cupsd.conf](images/04-cupsd-conf.png)

Després de modificar la configuració es reinicia el servei i es comprova que està actiu:

```bash
sudo systemctl restart cups
sudo systemctl status cups
```

L'estat és `active (running)` i el servei està `enabled`, per tant s'iniciarà automàticament amb el sistema.

![Reinici i estat de CUPS](images/05-restart-status-cups.png)

> **Nota de seguretat:** `Allow all` és còmode en un laboratori aïllat (xarxa Host-Only), però en un entorn real caldria limitar l'accés per adreça o subxarxa (per exemple `Allow 192.168.10.0/24`) i protegir la connexió amb xifrat.

---

## 4. Compartició de la impressora

### 4.1. Accés a la interfície web

Des del navegador del client Zorin OS s'accedeix a la interfície web de CUPS (versió 2.4.16) a `https://192.168.10.1:631`. El navegador mostra l'avís «No és segur» perquè el servidor utilitza un certificat autosignat, cosa normal en aquest entorn de proves.

![Pàgina d'inici de CUPS](images/06-cups-web-home.png)

### 4.2. Permisos d'administració

En entrar a `/admin`, CUPS demana autenticació. Perquè l'usuari `vboxuser` pugui administrar la impressora s'afegeix al grup `lpadmin` i es reinicia el servei:

```bash
sudo usermod -aG lpadmin vboxuser
sudo systemctl restart cups
```

![Autenticació a /admin](images/07-login-admin.png)

![Afegir vboxuser a lpadmin](images/08-usermod-lpadmin.png)

### 4.3. Afegir la impressora

A **Administració → Afegir impressora** CUPS detecta la impressora local `CUPS-PDF (Virtual PDF Printer)`. També mostra impressores de xarxa descobertes automàticament (una EPSON AM-C4000 de la xarxa on es troben les màquines virtuals), que no s'utilitzen en aquesta pràctica.

![Afegir impressora](images/09-add-printer-list.png)

Es configura la impressora amb les dades següents, activant l'opció **Compartir aquesta impressora**:

| Camp | Valor |
|---|---|
| Nom | `PDF_Printer` |
| Descripció | `Virtual PDF Printer` |
| Connexió | `cups-pdf:/` |
| Compartició | Activada |

![Nom, descripció i compartició](images/10-printer-name-sharing.png)

A l'assistent es va seleccionar la marca **Raw** com a driver.

![Selecció de marca Raw](images/11-select-make-raw.png)

### 4.4. Incidència i solució per línia de comandes

Durant la configuració va aparèixer l'error `cups-driver failed to get PPD file` (vegeu la part superior de la captura següent). Per resoldre-ho, es va crear la cua directament amb `lpadmin`, utilitzant el model `raw`, i es van activar l'acceptació i el processament de treballs:

```bash
sudo lpadmin -p PDF_Printer -v cups-pdf:/ -m raw -E
sudo cupsenable PDF_Printer
sudo cupsaccept PDF_Printer
```

| Opció | Significat |
|---|---|
| `-p PDF_Printer` | Nom de la impressora. |
| `-v cups-pdf:/` | URI del dispositiu (backend cups-pdf). |
| `-m raw` | Cua *raw*: els treballs s'envien sense processar. |
| `-E` | Habilita la impressora i accepta treballs. |
| `cupsenable` | Activa la impressora perquè processi treballs. |
| `cupsaccept` | Fa que la cua accepti nous treballs. |

CUPS avisa que les cues *raw* estan obsoletes i deixaran de funcionar en una versió futura. Funcionen a la versió actual (2.4.16), però és una limitació a tenir en compte.

![Creació de la cua amb lpadmin](images/13-lpadmin-raw-enable-accept.png)

---

## 5. Configuració del client Zorin OS

Des de **Configuració → Impressores → Afegir impressora** del client, es busca la impressora compartida pel servidor. Es pot indicar directament la URL del servidor IPP:

```
ipp://192.168.10.1
```

Una primera prova amb la ruta `ipp://192.168.10.1:631/printers/PDF_printers` no va trobar la impressora, perquè el nom correcte de la cua és `PDF_Printer` (sense la «s» final i respectant les majúscules).

![Primer intent amb URL](images/12-client-add-printer-url.png)

En escriure `ipp://192.168.10.1`, el client localitza el servidor i mostra la impressora **PDF_Printer** (adreça `192.168.10.1`) a la llista, juntament amb les impressores EPSON descobertes a la xarxa. Això demostra que el client pot localitzar la impressora compartida.

![PDF_Printer detectada al client](images/14-client-printer-detected.png)

---

## 6. Proves d'impressió

> **COMPLETAR:** afegiu aquí les captures i dades de les vostres proves.

Procediment:

1. Al client, obrir un document (text, imatge, PDF…) i triar **Imprimir → PDF_Printer**.
2. Repetir-ho amb diversos documents.
3. Comprovar l'estat de la cua des del client o des de la interfície web (pestanya **Treballs**).

Des del servidor també es poden consultar els treballs:

```bash
lpstat -t
lpstat -W completed -o
```

| Prova | Document | Resultat |
|---|---|---|
| 1 | *(nom del document)* | *(enviat / completat)* |
| 2 | *(nom del document)* | *(enviat / completat)* |
| 3 | *(nom del document)* | *(enviat / completat)* |

Captura suggerida: `images/15-print-test-client.png` i `images/16-jobs-cups-web.png`.

---

## 7. Comprovació dels documents generats

> **COMPLETAR:** afegiu aquí les captures de la comprovació al servidor.

Els fitxers PDF que genera `cups-pdf` es desen al servidor. Segons la configuració per defecte del paquet a Ubuntu, la ubicació és `/var/spool/cups-pdf/`, amb un subdirectori per usuari (o `ANONYMOUS` per a treballs anònims). La ruta exacta es pot verificar a `/etc/cups/cups-pdf.conf` (directiva `Out`).

```bash
grep -E "^(Out|AnonDirName)" /etc/cups/cups-pdf.conf
sudo ls -lR /var/spool/cups-pdf/
```

| Element | Dada |
|---|---|
| Ubicació dels fitxers | *(ruta comprovada)* |
| Documents creats | *(llista de PDF generats)* |
| Proves realitzades | *(resum de la secció 6)* |

Captura suggerida: `images/17-server-pdf-files.png`.

---

## 8. Conclusions

La prova de concepte mostra que CUPS permet muntar un servidor d'impressió complet amb pocs passos: instal·lar el servei, afegir una impressora virtual, obrir l'administració a la xarxa i compartir la cua perquè un client Linux la localitzi per IPP.

Aspectes que cal tenir en compte:

- **Seguretat:** la configuració utilitzada (`Allow all`, certificat autosignat) és adequada per a un laboratori, però en producció caldria restringir per xarxa i usuari.
- **Cues raw:** la cua es va crear com a *raw* després de l'error de PPD. CUPS ja avisa que aquest mode quedarà obsolet, de manera que una solució més duradora seria utilitzar el driver propi de cups-pdf.
- **Noms de cua:** un error petit en el nom (`PDF_printers` en lloc de `PDF_Printer`) impedeix que el client trobi la impressora; convé verificar el nom exacte amb `lpstat -p`.
