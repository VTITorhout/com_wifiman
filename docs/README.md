# Inleiding

Wanneer een toestel aangekocht wordt die verbinding moet maken met een privé netwerk is de manier van werken meestal als volgt:
1. Het  toestel start op in *acces point* (AP) mode en creëert hierbij een WiFi netwerk met een eigen SSID waarop *gebruikers* kunnen verbinden. Bij het verbinden met het netwerk krijgt de gebruiker een *landingspagina* voorgeschoteld (dit is de *captive portal*), waarop de essentiële instellingen kunnen gemaakt worden. Bij de meest essentiële instellingen behoort de keuze om te verbinden met het privé netwerk van de gebruiker met bijhorend wachtwoord. Optionele instellingen kunnen hier ook gemaakt worden.

![ESP32 als AP](./assets/esp32_ap.png)

2. Bij het opslaan van de instellingen herstart het toestel. Aangezien dit toestel nu voorzien is van instellingen start het niet meer op in _**a**cces **p**oint_ mode (AP), maar in _**sta**tion_ mode (STA). Hierbij probeert het toestel verbinding te maken met het privé netwerk door gebruik te maken van de nodige *credentials* die opgegeven zijn in vorige stap.
	* Slaagt het  toestel er in om een verbinding te maken, dan is de software voor instellingen en verbinding te maken niet meer van toepassing en kan overgeschakeld worden op de effectieve software van het toestel.
	* Slaagt het toestel er niet in om een verbinding te maken, dan is het meest logische om terug te herstarten in AP-mode zodat de gebruiker wijzigingen kan doorvoeren aan de instellingen.
	
![ESP32 als STA](./assets/esp32_sta.png)
	
Tijdens deze module worden de meest essentiële zaken aangehaald in verschillende stappen, gaande van basis tot expert.
1. [Basis configuratie](#basis-captive-portal) voor het *captive portal* om verbinding te maken met een privé netwerk
2. Opstarten van het *captive portal* op [aanvraag](#captive-portal-on-request) van de gebruiker
3. [Extra instellingen](#extra-instellingen) afvragen en opslaan
4. [Custom](#extra-instellingen) menu's / HTML
5. [Over The Air](#ota) upgraden van de firmware

# Installatie bibiliotheek

In principe zouden we alles van nodige software zelf kunnen schrijven, maar de gebruikers *tablatronix* en *tzapu* op GitHub hebben reeds een bibliotheek ontwikkeld die geschikt is voor dit doel. We gaan dan ook gebruik maken van deze bibliotheek.

In de Arduino omgeving kan via *schets* -> *Bibliotheek gebruiken* -> *Bibliotheken beheren* de nodige bibliotheek geïnstalleerd worden. Zoek hiervoor in de lijst "wifi manager" van de gebruiker *tablatronix*, en installeer vervolgens deze.

![Benodigde bibliotheek](./assets/arduino_bib.png)

Merk op dat deze bibliotheek zeer uitgebreid is en dat er momenteel nog veel wordt aan ontwikkeld. Op het moment van schrijven is er gebruik gemaakt van versie 2.0.13-beta, waarbij de beta betekent dat er opties aanwezig zijn die nog niet 100% werkende zijn bevonden. Deze opties kunnen in recentere versies anders werken of misschien zelf niet meer aanwezig zijn. Voor een lijst van mogelijkheden wordt altijd verwezen naar de [juiste *header*](https://github.com/tzapu/WiFiManager/blob/v2.0.13-beta/WiFiManager.h) file. Deze kan teruggevonden worden in de bibliotheek zelf (die op je systeem wordt gedownload) of in GitHub door langs boven de juiste *tag* te kiezen. 

![Correcte header file](./assets/bib_tag.png)

# Basis captive portal

Het meest essentiële onderdeel is het verbinden met een privé WiFi-netwerk. Hiervoor gaan we volgende basis code gebruiken:

```cpp
#include <WiFiManager.h>  //https://github.com/tzapu/WiFiManager

void setup(){
  Serial.begin(115200); //needed for debug, library will output a lot (due to beta version)
  WiFiManager portal; //set local, only needed in setup
  bool wifiConnected = portal.autoConnect("615-CaptivePortal"); //create blocking portal which tries to connect to AP if settings have been found
  if(!wifiConnected){
    //provided credentials does not result in connection to AP
    Serial.print("WiFi:\tUnable to connect to SSID ");
    Serial.println(portal.getWiFiSSID());
    portal.resetSettings();  //reset wrong credentials
    while(1); //no need to continue
  }
  Serial.print("WiFi:\tSuccesfully connected to SSID ");
  Serial.println(portal.getWiFiSSID());
  Serial.println("PRG:\tEntering loop");
}

void loop(){
  
}
```

Het zou kunnen zijn dat door toedoen van eerdere testen er reeds een SSID en wachtwoord zou opgeslagen zijn. Indien we niet verbonden geraken het de opgeslagen SSID worden de huidige *credentials* verwijderd en stopt de code met verder uit te voeren. Op het moment dat we vervolgens een reset doorvoeren (of de spannning kortstondig verwijderen) zal de ESP opstarten zonder *credentials* en zal het *captive portal* geactiveerd worden. 

In onderstaande afbeelding kan gezien worden wat er gebeurd in de seriële monitor wanneer er geen *credentials* zijn opgeslagen. Er wordt overgegaan van STA mode naar AP mode, waar vervolgens een nieuw netwerk wordt gecreëerd met de SSID *615-CaptivePortal*. De webserver voor het *captive portal* wordt gestart en er wordt gewacht op een gebruiker die verbinding maakt met het netwerk voor de nodige *credentials* in te voeren.

![Debug with no settings](./assets/dbg_01_no_settings.png)

Op het toestel die gebruikt wordt om verbinding te maken het netwerk is het volgende waar te nemen:

![Setup credentials](./assets/gsm_01.png)

1. Na het verbinden met het netwerk *615-CaptivePortal* komt een melding dat er verbinding is, maar dat er details te bekijken zijn. Dit is typisch bij een *captive portal* waar nog nood is aan een procedure om effectief verbonden te zijn.
2. Op de webpagina is te zien dat er geen AP instellingen zijn. We kiezen vervolgens voor *Configure WiFi* om de juiste instellingen door te voeren.
3. Langs boven wordt een lijst met alle beschikbare netwerken weergegeven alsook hun signaalsterkte. Klik op de netwerknaam om dit automatisch over te nemen in het SSID veld.
4. Vul vervolgens het correcte *password* in. Via *Show Password* kun je zien wat je invult. 
5. Klik vervolgens op *Save* om de instellingen op te slaan.
6. De code zal nu herstarten en proberen te verbinden met het gekozen netwerk. Als dit niet lukt zullen de instellingen gewist worden en bij herstarten zal je opnieuw aan het AP kunnen.

Als alles goed verloopt zul het het volgende te zien krijgen in de seriële monitor:

![Debug with saved settings](./assets/dbg_01_settings_saved.png)

Hier is duidelijk te zien dat er verbinding wordt gemaakt met de nieuwe SSID (die gekozen is in vorige stappen) en dat dit probleemloos lukt (de ESP ontvangt een IP). Het *captive portal* is nu niet meer nodig, en het toestel schakelt om naar STA mode. De uitkomtst van de *captive portal* is positief, dus kan er verder gegaan worden naar het programma van de gebruiker (in de loop).

Bij het resetten van het programma is het volgende te zien in de seriële monitor:

![Debug directly connected](./assets/dbg_01_connected.png)

De *captive portal* wordt nog niet gestart. Er wordt eerst geprobeerd verbinding te maken het de huidige *credentials*. Dit lukt hier binnen de twee seconden en meteen wordt overgegaan op het gebruikersprogramma (in de loop).

# Captive portal on request

Merk op dat bij vorig programma de *credentials* gewist werden wanneer er geen verbinding kon gemaakt worden. Dit is op zich goed om te testen, maar niet voor productie. Stel dat het netwerk even offline is, dan zal het toestel zijn *credentials* verwijderen, wat natuurlijk ongewenst is. 

Maar wat indien we ons vergist hebben van wachtwoord? Of dat we ons privé netwerk hebben aangepast (SSID gewijzigd, wachtwoord gewijzigd)? Er moet dus aan de code een aanpassing gebeuren dat we de instellingen terug kunnen oproepen. 

Onderstaande (herschreven) code bied hier een oplossing. Pin 23 (of andere, vrij te kiezen) wordt gebruikt om het *captive portal* op te roepen. Er is een functie geschreven die zich bezig houdt met het *captive portal*, en die kan aangeroepen worden met al dan niet een timeout. Indien er geen timeout opgegeven wordt, wordt gewoonweg geprobeerd om te verbinden met het netwerk met de huidige *credentials*. Indien wel een timeout wordt opgegeven wordt het *captive portal* gestart voor een bepaalde periode, zodat de gebruiker opnieuw kan verbinden om de instellingen te wijzigen. Het wijzigen kan het wissen of het aanpassen zijn van de *credentials*.

```cpp
#include <WiFiManager.h>  //https://github.com/tzapu/WiFiManager

#define GO_INTO_SETUP 23

bool wifiConnected; //to prevent user software to execute

bool setupWifi(uint16_t timeout = 0){
  WiFiManager portal; //create local instance of wifi manager
  portal.setEnableConfigPortal(false);  //prevent entering captive portal if connection failed, we would like to test credentials if timeout==0
  if(timeout){  //we need the captive portal
    portal.setConfigPortalTimeout(timeout); //how long to wait for setup?
    portal.setAPClientCheck(true); //when client is connected, portal can not timeout
    std::vector<const char *> portalMenu  = {"wifi","info","exit","sep","erase","update"};  //create menu with following possibilities, "sep" is seperator
    portal.setMenu(portalMenu);
    portal.startConfigPortal("615-CaptivePortal"); //create portal with timeout
  }
  bool test = portal.autoConnect(); //check if credentials are OK
  if(!test){
    //provided credentials does not result in connection to AP
    Serial.printf("WiFi:\tUnable to connect to SSID \"%s\"\r\n",portal.getWiFiSSID());
  }else{
    Serial.printf("WiFi:\tSuccesfully connected to SSID \"%s\"\r\n",portal.getWiFiSSID());
  }
  return test;  //return connection result to requester
}

void setup(){
  Serial.begin(115200); //needed for debug, library will output a lot (due to beta version)
  pinMode(GO_INTO_SETUP,INPUT_PULLUP);
  wifiConnected = setupWifi(); //just check if we get connected, settings are done on request
  Serial.println("PRG:\tEntering loop");
}

void loop(){
  if(wifiConnected){
    //user code
  }
  //check if we want to enter setup
  if(!digitalRead(GO_INTO_SETUP)){
    wifiConnected = setupWifi(60); //setup wifi with timeout of 60 seconds
  }
}
```

Het grote verschil zit hem bij het feit dat de code nu niet automatisch een *captive portal* zal opzetten als er geen verbinding kan gemaakt worden met het netwerk. 

![Debug no connection](./assets/dbg_02_no_settings.png)

Indien er geen *credentials* op het systeem worden aangetroffen wordt er meteen overgegaan naar het gebruikersprogramma. Het is nu aan de gebruiker om het *captive portal* op te roepen op aanvraag. Dit gebeurt door de gekozen pin laag te maken:

```cpp
//check if we want to enter setup
if(!digitalRead(GO_INTO_SETUP)){
  wifiConnected = setupWifi(60); //setup wifi with timeout of 60 seconds
}
```

![Debug request captive portal](./assets/dbg_02_request.png)

Net zoals bij het vorige programma kan op een identieke manier de *credentials* opgegeven worden via het *captive portal*. 

Het gedeelte rond het *captive portal* is echter wel herschreven. Er kan hier een extra parameter *timeout* meegegeven worden.
* Indien timeout = 0 wordt er enkel getest of de opgegeven *credentials* leiden tot een verbinding. Hiervan wordt gebruik gemaakt in de setup.
* Indien timeout > 0 wordt een *captive portal* gestart. Indien er geen verbinding met het *captive portal* binnen de opgegeven *timeout* wordt gemaakt wordt er teruggekeerd naar het gebruikersprogramma.
* Indien er een gebruiker verbinding maakt met het *captive portal* stopt de *timeout* zolang de gebruiker verbonden blijft. Dit kan (en zal) echter in sommige gevallen problemen met zich meebrengen. Zie hier voor het onderdeel [extra instellingen](#extra-instellingen).

Bovenstaande gebeurd door middel van volgende code:

```cpp
if(timeout){  //we need the captive portal
  portal.setConfigPortalTimeout(timeout); //how long to wait for setup?
  portal.setAPClientCheck(true); //when client is connected, portal can not timeout
  portal.startConfigPortal("615-CaptivePortal"); //create portal with timeout
}
```

Merk eveneens het volgende op in de code:

```cpp
std::vector<const char *> portalMenu  = {"wifi","info","exit","sep","erase","update"};  //create menu with following possibilities, "sep" is seperator
portal.setMenu(portalMenu);
```

De layout van het *captive portal* kan bevolkt en herschikt worden naar believen. Het bestand `strings_en.h` die in de bibliotheek terug te vinden is *beschrijft* alle mogelijkheden. Er wordt hier expliciet verwezen naar de code aangezien de documentatie momenteel te wensen overlaat:

```cpp
const char * const HTTP_PORTAL_MENU[] PROGMEM = {
"<form action='/wifi'    method='get'><button>Configure WiFi</button></form><br/>\n", // MENU_WIFI
"<form action='/0wifi'   method='get'><button>Configure WiFi (No Scan)</button></form><br/>\n", // MENU_WIFINOSCAN
"<form action='/info'    method='get'><button>Info</button></form><br/>\n", // MENU_INFO
"<form action='/param'   method='get'><button>Setup</button></form><br/>\n",//MENU_PARAM
"<form action='/close'   method='get'><button>Close</button></form><br/>\n", // MENU_CLOSE
"<form action='/restart' method='get'><button>Restart</button></form><br/>\n",// MENU_RESTART
"<form action='/exit'    method='get'><button>Exit</button></form><br/>\n",  // MENU_EXIT
"<form action='/erase'   method='get'><button class='D'>Erase</button></form><br/>\n", // MENU_ERASE
"<form action='/update'  method='get'><button>Update</button></form><br/>\n",// MENU_UPDATE
"<hr><br/>" // MENU_SEP
};
```

* `wifi`: mogelijkheid om te verbinden met SSID en wachtwoord op te geven. Bij het openen wordt er gescaned naar de beschikbare netwerken die vervolgens in een lijst worden opgenomen.
* `info`: allerlei informatie over het netwerk, systeem en bibliotheek.
* `exit`: de *captive portal* wordt hierdoor afgesloten.
* `sep`: separator; scheidingslijn.
* `erase`: wis alle instellingen van de WiFi (SSID & paswoord).
* `update`: mogelijkheid om een nieuw hex bestand in te laden. Op deze manier kan de *firmware* gewijzigd worden zonder dat er gebruik moet gemaakt worden van de Arduino IDE. Let op; dit werkt enkel maar goed op een *echte* browser.

Merk op dat er nog extra mogelijkheden zijn:
* `0wifi`: identiek als `wifi`, maar dan zonder het scannen tijdens het openen. Hier moet je manueel de SSID opgeven.
* `param`: mogelijkheid tot [extra instellingen](#extra-instellingen).
* `close`: het menu van de *captive portal* wordt hierdoor verlaten, maar de *captive portal* blijft actief.
* `restart`: de ESP wordt hierdoor herstart.

# Extra instellingen

De *WiFi Manager* biedt ook de mogelijkheid om *custom parameters* door te geven. Indien ons toestel naast verbinding met het WiFi netwerk ook nog andere verbindingen moeten maken over het internet, met bijvoorbeeld *cloud* toepassingen, moeten ook hiervoor *credentials* kunnen opgegeven worden.

In het onderdeel [captive portal on request](#captive-portal-on-request) was reeds vermelding gemaakt van `param` die aan het menu kan toegevoegd worden. Deze mogelijkheid in de bibliotheek creëert een extra pagina waarop gebruikerparameters kunnen geplaatst worden. Het toevoegen van gebruikerparameters kan vervolgens gebeuren door het commando `bool addParameter(WiFiManagerParameter *p);`, waarbij `WiFiManagerParameter` als volgt moet opgebouwd worden:

```cpp
/** 
  Create custom parameters that can be added to the WiFiManager setup web page
  @id is used for HTTP queries and must not contain spaces nor other special characters
*/
WiFiManagerParameter();
WiFiManagerParameter(const char *custom);
WiFiManagerParameter(const char *id, const char *label);
WiFiManagerParameter(const char *id, const char *label, const char *defaultValue, int length);
WiFiManagerParameter(const char *id, const char *label, const char *defaultValue, int length, const char *custom);
WiFiManagerParameter(const char *id, const char *label, const char *defaultValue, int length, const char *custom, int labelPlacement);
```

Hierbij zijn volgende zaken van belang:
* `id`: unieke ID waarmee de parameter kan benaderd worden (in HTML).
* `label`: tekst die voor het gebruikersveld moet geplaatst worden.
* `defaultValue`: inhoud van het veld, kan gebruikt worden indien de *setting* die reeds in het geheugen zit moet weergegeven worden.
* `length`: maximale lengte van het veld
* `custom`: additionele HTML code die toegevoegd moet worden, kan gebruikt worden om bijvoorbeeld [*regular expressions*](https://en.wikipedia.org/wiki/Regular_expression) toe te voegen.

Als voorbeeld wordt het gebruik van een MQTT verbinding genomen. Om deze verbinding te kunnen realiseren is er nood aan een serveradres, een poort en al dan niet een gebruikersnaam en wachtwoord. Voor de MQTT code wordt verwezen naar de module [MQTT](https://innovet-mqtt.netlify.app/#esp32) die eveneens terug te vinden is op [stem-ict.be](https://stem-ict.be).

```cpp
WiFiManagerParameter custom_mqtt_server("server", "MQTT server", mqtt.server, 60,"pattern='([\\w_-]+(?:(?:\\.[\\w_-]+)+))([\\w.,@?^=%&:\\/~+#-]*[\\w@?^=%&\\/~+#-])'");
WiFiManagerParameter custom_mqtt_port("port", "MQTT port", mqtt.port, 5,"pattern='\\d{1,5}'");
WiFiManagerParameter custom_mqtt_user("user", "MQTT user", mqtt.user, 30);
WiFiManagerParameter custom_mqtt_pass("pass", "MQTT password", mqtt.password, 20,"type='password'");
```

Bovenstaande code beschrijft de additionele parameter die nodig zijn voor de MQTT verbinding. Merk op dat er gebruik is gemaakt van extra HTML code die de invoer van de velden controleert of er een wachtwoord veld van maakt.

Indien we ook `param` toevoegen aan het menu kunnen we met deze pagina de 4 velden weergeven en aanpassen. Vergeet wel niet ook de extra parameters aan de *wifi manager* door te geven.

```cpp
portal.addParameter(&custom_mqtt_server);
portal.addParameter(&custom_mqtt_port);
portal.addParameter(&custom_mqtt_user);
portal.addParameter(&custom_mqtt_pass);
std::vector<const char *> portalMenu  = {"wifi","param","info","exit","sep","erase","update"};  //create menu with following possibilities, "sep" is seperator
portal.setMenu(portalMenu);
``` 

De `param` pagina is iets die geleidelijk aan bij de code is toegevoegd. De interactie met de gebuiker is hiervoor niet optimaal, en het opslaan van de parameters gebeurd niet door de bibiliotheek, maar moet de gebruiker zelf voor instaan. Er kunnen hiervoor wel *callbacks* ingevoerd worden. Dit zijn *jumps* naar specifieke gebruikerscode die toelaten tussendoor extra zaken uit te voeren.

```cpp
portal.setPreSaveParamsCallback(saveParamsCallback);  //needed to trigger save
```

Ergens in de code moet dan de subroutine `saveParamsCallback` terug te vinden zijn. In die subroutine zouden we de parameters meteen kunnen opslaan, maar de code wordt geschreven dat de wifi manager en alles die er bij hoort enkel maar in de *scope* `setupWifi` bestaat. Hierdoor kunnen we de parameters niet benaderen vanuit een andere *scope*, en zal dit op een andere manier moeten opgelost worden. De oplossing bestaat er in een *boolean* te setten die *globaal* bestaat zodat deze ook kan geraadpleegd worden in een andere *scope*. 

```cpp
bool shouldSaveParams = false;
void saveParamsCallback () {
  shouldSaveParams = true;
}
```

in de `setupWifi` controleren we vervolgens of deze *boolean* geset is en voeren we de noodzakelijke acties uit:

```cpp
if(shouldSaveParams){ //params have been update, continue
  //retrieve values
  strcpy(mqtt.server, custom_mqtt_server.getValue());
  strcpy(mqtt.port, custom_mqtt_port.getValue());
  strcpy(mqtt.user, custom_mqtt_user.getValue());
  strcpy(mqtt.password, custom_mqtt_pass.getValue());
  Serial.printf("MQTT:\tWill save next params\r\n\tServer: %s\r\n\tPort: %s\r\n\tUser: %s\r\n\tPassword: %s\r\n",mqtt.server,mqtt.port,mqtt.user,mqtt.password);
  ...
  shouldSaveParams = false;
}
```

Het grote probleem die dan optreedt is dat we gebruik wensen te maken van een timeout, en zolang er iemand verbonden blijft met het *captive portal* zal er geen timeout optreden en blijft de code dan ook hangen bij de routine `portal.startConfigPortal("615-CaptivePortal");`. De `if` functie wordt nooit bereikt. De oplossing bestaat er nu in het *captive portal* niet te gebruiken als *blocking code*, maar zelf deze te *processen*. Daarvoor passen we de code aan als volgt:

```cpp
uint32_t portalStarted = millis();
portal.setConfigPortalBlocking(false);  //we will process the portal by ourselves
portal.startConfigPortal("615-CaptivePortal");
while((portalStarted+timeout*1000)>millis()){
  portal.process(); //there is time left
}
```

Hierbij is het jammergenoeg onmogelijk te controleren of er een gebruiker verbonden is met het *captive portal*, want indien er een verbinding is moet er geen timeout optreden. Om dit op te lossen zullen we de bibliotheek **manueel** moeten aanpassen. In de bibliotheek is de *private* functie `uint8_t WiFi_softap_num_stations();` opgenomen. Als we deze *public* plaatsen kunnen we dit gebruiken om de timeout te resetten:

```cpp
if(portal.WiFi_softap_num_stations()>0){ //there are clients connected, reset timeout
  portalStarted = millis(); //reset current time
}
```

Eerder werd aangehaald dat het aan de gebruiker is om de extra parameters ook zelf te verwerken. De SSID en het paswoord van de WiFi worden door de bibliotheek opgeslagen in NVRAM/flash. De extra parameters worden nergens opgeslagen. Ook deze extra parameters wensen we te behouden tijdens onderbrekingen van de spanning. Hiervoor moeten we zelf de parameters opslaan in NVRAM/flash. 

De eenvoudigste manier om dit te bereiken is a.d.h.v. de bibliotheek *preferences* die standaard opgenomen is in de Arduino omgeving voor de ESP processoren. In oudere versies moest gebruik gemaakt worden van de bibliotheek *EEPROM*, maar deze is ondertussen *obsolete* geworden. 

De bibliotheek *preferences* laat toe een *namespace* te openen waar verschillende variabelen kunnen gestockeerd worden. Dit gebeurd als volgt:
```cpp
#include <Preferences.h>  //needed to save force parameters
Preferences pref;  //create a preference object
pref.begin("mqtt",false); //start namespace "mqtt" in R/W mode
...
pref.end();
```

Merk op dat voor de naam van de *namespace* een willekeurige naam kan gekozen worden, maar dat deze in lengte gelimiteerd is tot 15 karakters. Hier is geopteerd voor *mqtt* aangezien we de connectiegegevens van een MQTT server gaan opslaan.

Op de plaats van de `...` kunnen we de variabelen benaderen als een *key-value* paar. Dit stemt heel goed overeen met hoe een JSON object is opgebouwd. Voor de mogelijkheden van de data formaten wordt verwezen naar de [*readthedocs*](https://espressif-docs.readthedocs-hosted.com/projects/arduino-esp32/en/latest/api/preferences.html) van de bibliotheek.

Aangezien de variabelen zich in NVRAM/flash bevinden moeten we deze van en naar het werkgeheugen (RAM) verplaatsen vooraleer we deze kunnen bewerken. Dit gebeurd a.d.h.v. `getString(key)` en `putString(key,value)`. Om de variabelen te groeperen maken we gebruik van een *struct* waarin de variabelen passen.

```cpp
struct mqttSettings{
  char server[60];
  char port[6];
  char user[30];
  char password[20];
};
mqttSettings mqtt;
```

Tijdens het starten van het programma laden we de *namespace* mqtt en halen we de variabelen op uit NVRAM/flash a.d.h.v. hun key. We plaatsen deze meteen in onze *struct*:
```cpp
pref.begin("mqtt",false); //start namespace "mqtt" in R/W mode
  strcpy(mqtt.server,pref.getString("server","").c_str());
  strcpy(mqtt.port,pref.getString("port","").c_str());
  strcpy(mqtt.user,pref.getString("user","").c_str());
  strcpy(mqtt.password,pref.getString("password","").c_str());
  Serial.printf("MQTT:\tLoaded next params\r\n\tServer: %s\r\n\tPort: %s\r\n\tUser: %s\r\n\tPassword: %s\r\n",mqtt.server,mqtt.port,mqtt.user,mqtt.password);
pref.end();
```

Het stukje code `.c_str()` na het ophalen van de *string* zet deze om naar een *null-terminated character array*. Het is beter gebruik te maken van in lengte gelimiteerde *character arrays* dan van *strings* die mogelijk voor gefragmenteerd geheugen zullen zorgen.

Eenzelfde iets voeren we uit wanneer de gebruiker nieuwe waarden opslaat via de *param* pagina:

```cpp
//retrieve values
  strcpy(mqtt.server, custom_mqtt_server.getValue());
  strcpy(mqtt.port, custom_mqtt_port.getValue());
  strcpy(mqtt.user, custom_mqtt_user.getValue());
  strcpy(mqtt.password, custom_mqtt_pass.getValue());
//begin saving values
  pref.begin("mqtt",false); //start namespace "mqtt" in R/W mode
  Serial.printf("MQTT:\tWill save next params\r\n\tServer: %s\r\n\tPort: %s\r\n\tUser: %s\r\n\tPassword: %s\r\n",mqtt.server,mqtt.port,mqtt.user,mqtt.password);
    pref.putString("server",mqtt.server);
    pref.putString("port",mqtt.port);
    pref.putString("user",mqtt.user);
    pref.putString("password",mqtt.password);
  pref.end();
  Serial.println("MQTT:\tSettings have been saved!");
```

De ingegeven waarden worden opgeslagen via de URL, dit a.d.h.v. een [POST](https://en.wikipedia.org/wiki/POST_(HTTP)). Deze komen dus niet in een variabele in de code terecht, maar moeten opgevraagd worden via de bibliotheek. De waarde moet hierna gekopieerd worden naar de *struct* zodat er verder mee kan gewerkt worden. 

Eenmaal deze in de struct terecht zijn gekomen worden ze verplaatst naar NVRAM/flash.

De totale code zou er dan als volgt kunnen uitzien:

```cpp
#include <WiFiManager.h>  //https://github.com/tzapu/WiFiManager
#include <Preferences.h>  //needed to save force parameters

#define GO_INTO_SETUP 23

Preferences pref;  //

struct mqttSettings{
  char server[60];
  char port[6];
  char user[30];
  char password[20];
};
mqttSettings mqtt;

bool wifiConnected; //to prevent user software to execute

void loadConfig(){
  pref.begin("mqtt",false); //start namespace "mqtt" in R/W mode
  strcpy(mqtt.server,pref.getString("server","").c_str());
  strcpy(mqtt.port,pref.getString("port","").c_str());
  strcpy(mqtt.user,pref.getString("user","").c_str());
  strcpy(mqtt.password,pref.getString("password","").c_str());
  Serial.printf("MQTT:\tLoaded next params\r\n\tServer: %s\r\n\tPort: %s\r\n\tUser: %s\r\n\tPassword: %s\r\n",mqtt.server,mqtt.port,mqtt.user,mqtt.password);
  pref.end();
}

bool shouldSaveParams = false;
bool shouldClosePortal = false;

void saveParamsCallback () {
  shouldSaveParams = true;
}

void saveWifiCallback () {
  shouldClosePortal = true;
}

bool setupWifi(uint16_t timeout = 0){
  WiFiManager portal; //create local instance of wifi manager
  portal.setEnableConfigPortal(false);  //prevent entering captive portal if connection failed, need this just to test if connection could be made (timeout=0)
  if(timeout){  //we need the captive portal
    uint32_t portalStarted = millis();
    std::vector<const char *> portalMenu  = {"wifi","param","info","exit","sep","erase","update"};  //create menu with following possibilities, "sep" is seperator
    portal.setMenu(portalMenu);
    WiFiManagerParameter custom_mqtt_server("server", "MQTT server", mqtt.server, 60,"pattern='([\\w_-]+(?:(?:\\.[\\w_-]+)+))([\\w.,@?^=%&:\\/~+#-]*[\\w@?^=%&\\/~+#-])'");
    WiFiManagerParameter custom_mqtt_port("port", "MQTT port", mqtt.port, 5,"pattern='\\d{1,5}'");
    WiFiManagerParameter custom_mqtt_user("user", "MQTT user", mqtt.user, 30);
    WiFiManagerParameter custom_mqtt_pass("pass", "MQTT password", mqtt.password, 20,"type='password'");
    portal.addParameter(&custom_mqtt_server);
    portal.addParameter(&custom_mqtt_port);
    portal.addParameter(&custom_mqtt_user);
    portal.addParameter(&custom_mqtt_pass);
    portal.setPreSaveParamsCallback(saveParamsCallback);  //needed to trigger save
    portal.setPreSaveConfigCallback(saveWifiCallback);  //needed to close portal
    portal.setConfigPortalBlocking(false);  //we will process the portal by ourselves
    portal.startConfigPortal("615-CaptivePortal");
    while((portalStarted+timeout*1000)>millis()){
      portal.process(); //there is time left
      if(portal.WiFi_softap_num_stations()>0){ //there are clients connected, reset timeout
        portalStarted = millis(); //reset current time
      }
      if(shouldSaveParams){ //params have been update, continue
        //retrieve values
        strcpy(mqtt.server, custom_mqtt_server.getValue());
        strcpy(mqtt.port, custom_mqtt_port.getValue());
        strcpy(mqtt.user, custom_mqtt_user.getValue());
        strcpy(mqtt.password, custom_mqtt_pass.getValue());
        //begin saving values
        pref.begin("mqtt",false); //start namespace "mqtt" in R/W mode
        Serial.printf("MQTT:\tWill save next params\r\n\tServer: %s\r\n\tPort: %s\r\n\tUser: %s\r\n\tPassword: %s\r\n",mqtt.server,mqtt.port,mqtt.user,mqtt.password);
        pref.putString("server",mqtt.server);
        pref.putString("port",mqtt.port);
        pref.putString("user",mqtt.user);
        pref.putString("password",mqtt.password);
        pref.end();
        Serial.println("MQTT:\tSettings have been saved!");
        shouldSaveParams = false;
        shouldClosePortal = true;
      }
      if(shouldClosePortal){
        portal.stopWebPortal();
        shouldClosePortal = false;
        break;  //break the while
      }
    }
  }
  bool test = portal.autoConnect(); //check if credentials are OK
  if(!test){
    //provided credentials does not result in connection to AP
    Serial.printf("WiFi:\tUnable to connect to SSID \"%s\"\r\n",portal.getWiFiSSID());
  }else{
    Serial.printf("WiFi:\tSuccesfully connected to SSID \"%s\"\r\n",portal.getWiFiSSID());
  }
  return test;  //return connection result to requester
}

void setup(){
  Serial.begin(115200); //needed for debug, library will output a lot (due to beta version)
  pinMode(GO_INTO_SETUP,INPUT_PULLUP);
  loadConfig(); //load settings of MQTT
  wifiConnected = setupWifi(); //just check if we get connected, settings are done on request
  Serial.println("PRG:\tEntering loop");
}

void loop(){
  if(wifiConnected){
    //user code
  }
  //check if we want to enter setup
  if(!digitalRead(GO_INTO_SETUP)){
    wifiConnected = setupWifi(60); //setup wifi with timeout of 60 seconds
  }
}
```

# OTA
