# noVNC Integratie Uitleg

## 1. Overzicht

Dit document beschrijft hoe een op afstand bestuurde browser (draaiend via Playwright binnen een Docker-container) wordt weergegeven in de frontend van de applicatie met behulp van noVNC. Het doel is om de gebruiker een visuele interface te bieden van de browser waarmee de geautomatiseerde agent interacteert.

## 2. Python Backend (`VncDockerPlaywrightBrowser`)

De `VncDockerPlaywrightBrowser` klasse in de Python backend is verantwoordelijk voor het opzetten en beheren van een Playwright-browserinstantie die draait binnen een Docker-container die ook een VNC-server bevat.

*   **Poortbepaling (`novnc_port` en `playwright_port`)**:
    *   De klasse accepteert standaardwaarden voor `novnc_port` (standaard 6080) en `playwright_port` (standaard 37367) tijdens initialisatie.
    *   De methode `_generate_new_browser_address` wordt gebruikt (hoewel de aanroep ervan niet expliciet getoond wordt in de constructor, suggereert de logica dat dit bedoeld is om poortconflicten te vermijden). Deze methode roept `_get_available_port` aan, die een willekeurige beschikbare poort op de hostmachine vindt en toewijst aan `self._playwright_port` en `self._novnc_port`. Dit zorgt ervoor dat, indien de standaardpoorten bezet zijn, er alternatieve poorten worden gebruikt.
    *   De `_hostname` wordt geconfigureerd als `localhost` als de client niet binnen Docker draait, of een specifiekere naam gebaseerd op de poorten en het websocket-pad als dat wel het geval is.

*   **Rol van de `magentic-ui-vnc-browser` Docker-image**:
    *   Deze Docker-image (standaard `magentic-ui-vnc-browser`, maar kan worden overschreven) is cruciaal. Het bevat de benodigde software:
        *   Een besturingssysteem met een window manager.
        *   Een VNC-server (bijv. x11vnc, TigerVNC).
        *   noVNC: een webgebaseerde VNC-client.
        *   Playwright en de benodigde browsers (bijv. Chromium).
        *   Een entrypoint-script of server (zoals de `playwright-server.js` vermeld in de Dockerfile van `magentic-ui-browser-docker` die een vergelijkbaar doel dient) dat de Playwright-server en noVNC-server start op basis van de ontvangen omgevingsvariabelen.

*   **Poortmapping van Host naar Container**:
    *   Bij het aanmaken van de Docker-container (`create_container` methode) worden de hostpoorten gemapt naar de containerpoorten:
        ```python
        ports={
            f"{self._playwright_port}/tcp": self._playwright_port, # Host-poort : Container-poort (impliciet gelijk)
            f"{self._novnc_port}/tcp": self._novnc_port,       # Host-poort : Container-poort (impliciet gelijk)
        }
        ```
        Dit betekent dat verkeer naar `localhost:<host_playwright_port>` wordt doorgestuurd naar de Playwright-server in de container, en verkeer naar `localhost:<host_novnc_port>` wordt doorgestuurd naar de noVNC-server in de container. De container zelf moet zijn Playwright- en noVNC-services configureren om op deze respectievelijke poortnummers intern te luisteren.

*   **Omgevingsvariabelen en hun rol**:
    De volgende omgevingsvariabelen worden doorgegeven aan de Docker-container bij het starten:
    *   `PLAYWRIGHT_WS_PATH`: Een (optioneel willekeurig) pad voor de Playwright WebSocket-verbinding.
    *   `PLAYWRIGHT_PORT`: Het poortnummer waarop de Playwright-server *binnen de container* moet luisteren. Dit komt overeen met de `self._playwright_port` van de host.
    *   `NO_VNC_PORT`: Het poortnummer waarop de noVNC-webserver *binnen de container* moet luisteren. Dit komt overeen met de `self._novnc_port` van de host.
    *   **Verondersteld gebruik binnen de container**:
        *   Het entrypoint-script of de server binnen de Docker-image gebruikt `PLAYWRIGHT_PORT` en `PLAYWRIGHT_WS_PATH` om de Playwright-browserserver te starten en toegankelijk te maken.
        *   Het gebruikt `NO_VNC_PORT` om de noVNC-service te configureren, die de VNC-sessie van de browser via het web aanbiedt.

*   **Resulterende `vnc_address`**:
    *   De `VncDockerPlaywrightBrowser` klasse stelt een `vnc_address` property beschikbaar.
    *   Deze wordt geconstrueerd als `f"http://{self._hostname}:{self._novnc_port}/vnc.html"`. Gezien `_hostname` meestal `localhost` is (tenzij de client zelf in Docker draait en een specifieke netwerkconfiguratie gebruikt), is de typische URL `http://localhost:<novnc_port>/vnc.html`.

## 3. Frontend (`BrowserIframe.tsx`)

De `BrowserIframe.tsx` React-component is verantwoordelijk voor het daadwerkelijk weergeven van de noVNC-interface in de frontend.

*   **Ontvangst van `novncPort`**:
    *   De component ontvangt de `novncPort` als een prop. Deze poort is de hostpoort die is gemapt naar de noVNC-service in de Docker-container (zoals bepaald door de `VncDockerPlaywrightBrowser` backend).

*   **Constructie van `vncUrl`**:
    *   De component bouwt de `vncUrl` dynamisch op:
        ```javascript
        const vncUrl = `http://localhost:${novncPort}/vnc.html?autoconnect=true&resize=${
          scaling === "remote" ? "remote" : "scale"
        }&show_dot=true&scaling=${scaling}&quality=${quality}&compression=0&view_only=${
          viewOnly ? 1 : 0
        }`;
        ```
    *   Cruciaal hier is het gebruik van `localhost:${novncPort}`. Dit verwijst naar de hostmachine vanuit het perspectief van de browser van de gebruiker. Omdat de Docker-poort is gemapt, zal een verzoek aan `localhost` op de hostmachine op de `novncPort` worden doorgestuurd naar de noVNC-service die in de Docker-container draait.
    *   Diverse queryparameters (`autoconnect`, `resize`, `scaling`, `quality`, `view_only`) worden toegevoegd om het gedrag van de noVNC-client aan te passen.

*   **Gebruik van `<iframe>`**:
    *   De component rendert een `<iframe>` HTML-element.
    *   De `src`-attribuut van dit iframe wordt ingesteld op de geconstrueerde `vncUrl`.
    *   Dit zorgt ervoor dat de noVNC-webclientinterface wordt geladen binnen het iframe. noVNC maakt vervolgens verbinding met de VNC-server die de daadwerkelijke browser-UI streamt vanuit de Docker-container.

## 4. Verbindingsstroom (Samenvatting)

1.  Een gebruiker initieert een actie in de frontend die de visualisatie van een door Playwright bestuurde browser vereist.
2.  De Python backend (specifiek een instantie van `VncDockerPlaywrightBrowser`) wordt aangeroepen.
3.  De `VncDockerPlaywrightBrowser`-instantie bepaalt de te gebruiken hostpoorten voor Playwright (`playwright_port`) en noVNC (`novnc_port`), mogelijk dynamisch om conflicten te voorkomen.
4.  De backend start de `magentic-ui-vnc-browser` Docker-container. Hierbij worden de gekozen `playwright_port` en `novnc_port`, samen met het `PLAYWRIGHT_WS_PATH`, als omgevingsvariabelen (`PLAYWRIGHT_PORT`, `NO_VNC_PORT`, `PLAYWRIGHT_WS_PATH`) doorgegeven aan de container. De Docker-poortmapping wordt ingesteld om `host:<playwright_port>` -> `container:<PLAYWRIGHT_PORT>` en `host:<novnc_port>` -> `container:<NO_VNC_PORT>` te koppelen.
5.  Binnen de Docker-container:
    *   Een Playwright-server start en luistert op de interne `PLAYWRIGHT_PORT` op het `PLAYWRIGHT_WS_PATH`.
    *   Een VNC-server streamt de desktop/browser UI.
    *   Een noVNC-server start en luistert op de interne `NO_VNC_PORT`, en dient de webgebaseerde VNC-client. Deze noVNC-server is geconfigureerd om verbinding te maken met de lokale VNC-server binnen dezelfde container.
6.  De frontend (React-applicatie) ontvangt het `novnc_port` nummer (de hostpoort) van de backend.
7.  De `BrowserIframe` component in de frontend gebruikt deze `novnc_port` om de URL `http://localhost:<novnc_port}/vnc.html` te construeren.
8.  Het `<iframe>` in `BrowserIframe` laadt deze URL.
9.  De browser van de eindgebruiker maakt (via het iframe) verbinding met de noVNC-webserver die draait *binnen de Docker-container* (toegankelijk gemaakt via de poortmapping op `localhost:<novnc_port>`).
10. De noVNC-interface wordt geladen in het iframe, die op zijn beurt de externe browser-UI weergeeft, waardoor de gebruiker de geautomatiseerde browsersessie kan zien en eventueel besturen (indien niet `viewOnly`).

Deze stroom maakt het mogelijk om een geïsoleerde, programmatisch bestuurde browser visueel toegankelijk te maken via een standaard webinterface.
