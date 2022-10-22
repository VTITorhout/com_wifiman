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
4. [Custom](#custom-menus--html) menu's / HTML

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

WiFiManager portal; //set global, so we can access on request

void setup(){
  Serial.begin(115200); //needed for debug, library will output a lot (due to beta version)
  bool test = portal.autoConnect("615-CaptivePortal"); //create blocking portal which tries to connect to AP if settings have been found
  if(!test){
    //provided credentials does not result in connection to AP
    Serial.print("WiFi:\tUnable to connect to SSID ");
    Serial.println(portal.getWiFiSSID());
	portal.resetSettings();	//reset wrong credentials
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

De *captive portal* wordt nog niet gestart. Er wordt eerst geprobeerd verbinding te maken het de huidige *credentials* Dit lukt binnen de twee seconden en meteen wordt overgegaan op het gebruikersprogramma (in de loop).

# Captive portal on request

Dit is op zich goed om te testen, maar niet voor productie. Stel dat het netwerk even offline is, dan zal het toestel zijn *credentials* verwijderen, wat natuurlijk ongewenst is.

# Extra instellingen
# Custom menu's & HTML
# OTA