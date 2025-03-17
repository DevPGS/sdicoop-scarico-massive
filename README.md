# sdicoop-scarico-massive
Istruzioni per scarico fatture da sdi servizio massivo

Vedi anche [Servizi massivi SDI Coop](https://www.agenziaentrate.gov.it/portale/servizi-massivi-sdicoop)

Topic di partenza [Forum Italia - Invio richiesta di Download massivo al servizio web](https://forum.italia.it/t/invio-richiesta-di-download-massivo-al-servizio-web/43307)

## compilo questo template:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ns1:InputMassivo
 xsi:schemaLocation="http://www.sogei.it/InputPubblico https://www.fatturapa.gov.it/export/documenti/ws/servizimassivi/InputMassivo_v1.4.xsd"
 xmlns:ns1="http://www.sogei.it/InputPubblico"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
>
  <ns1:TipoRichiesta>
    <ns1:Fatture>
      <ns1:Richiesta>FATT</ns1:Richiesta>
      <ns1:ElencoPiva>
        <ns1:Piva>QUI_LA_PARTITA_IVA</ns1:Piva>
      </ns1:ElencoPiva>
      <ns1:TipoRicerca>PUNTUALE</ns1:TipoRicerca>
      <ns1:FattureRicevute>
        <ns1:DataEmissione>
          <ns1:Da>2025-01-01</ns1:Da>
          <ns1:A>2025-01-31</ns1:A>
        </ns1:DataEmissione>
        <ns1:Flusso><ns1:Tutte>ALL</ns1:Tutte></ns1:Flusso>
        <ns1:Ruolo>CESSIONARIO</ns1:Ruolo>
      </ns1:FattureRicevute>
    </ns1:Fatture>
  </ns1:TipoRichiesta>
</ns1:InputMassivo>
```

## verifico la sintassi con 
`xmllint --noout --schema schema/InputMassivo_v1.4.xsd inputmassivo.xml` (linux)

## converto la richiesta xml in base64

`base64 inputmassivo.xml > inputmassivo.base64` (linux)

`certutil -encode inputmassivo.xml inputmassivo.base64` (windows)

su windows eliminare -- BEGIN CERTIFICATE -- END CERTIFICATE ---

## inserisco il base64 generato in questo template
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ns1:FileRichiesta xmlns:ns1="http://ivaservizi.agenziaentrate.gov.it/docs/xsd/ServiziMassivi/input/RichiestaServiziMassivi/v1.0"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://ivaservizi.agenziaentrate.gov.it/docs/xsd/ServiziMassivi/input/RichiestaServiziMassivi/v1.0 RichiestaServiziMassivi_v1.0.xsd"
 versione="1.0"
>
  <TipoRichiesta>FATT</TipoRichiesta>
  <NomeFile>RichiestaMassiva.xml</NomeFile>
  <File>QUI_IL_BASE64_GENERATO</File>
</ns1:FileRichiesta>
```

## verifico la sintassi con 
`xmllint --noout --schema schema/RichiestaServiziMassivi_v1.0.xsd dafirmare.xml` (linux)

## firmo la richiesta con questo comando
`openssl smime -sign -in dafirmare.xml -out firmato.xml.p7m -signer /cert/file.cer -inkey /cert/client.key -outform DER -nodetach`

> solo come prova, in realtà in produzione va firmato con un firma elettronica valida (aruba / infocert / ecc...).
> Inviando il file firmato con questo metodo, viene generata una notifica firma non valida.

## converto in base64 la richiesta firmata
`base64 firmato.xml.p7m > firmato.base64` (linux)

`certutil -encode firmato.xml.p7m firmato.base64` (windows)

## inserisco il base64 firmato in questo template
``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:typ="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
  <soapenv:Header/>
  <soapenv:Body>
    <typ:InoltroRichiestaRequest>
      <ns1:FileRichiesta xmlns:ns1="http://ivaservizi.agenziaentrate.gov.it/docs/xsd/ServiziMassivi/input/RichiestaServiziMassivi/v1.0"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://ivaservizi.agenziaentrate.gov.it/docs/xsd/ServiziMassivi/input/RichiestaServiziMassivi/v1.0
       RichiestaServiziMassivi_v1.0.xsd" versione="1.0"
      >
        <TipoRichiesta>FATT</TipoRichiesta>
	      <NomeFile>RichiestaMassiva.xml.p7m</NomeFile>
	      <File>QUI_IL_BASE64_DEL_FILE_FIRMATO</File>
      </ns1:FileRichiesta>
    </typ:InoltroRichiestaRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

## invio la richiesta tramite soap
https://servizi.fatturapa.it/sm-scarico-file -> inoltroRichiesta

La risposta sarà tipo questa

``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
   <soapenv:Body>
      <a:InoltroRichiestaResponse xmlns:a="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
         <IdRichiesta>62609XXX</IdRichiesta>
         <DataOraRicezione>2025-03-17T15:01:40.019+01:00</DataOraRicezione>
      </a:InoltroRichiestaResponse>
   </soapenv:Body>
</soapenv:Envelope>
```

## esito richiesta
Per sapere se è andata a buon fine bisognerà chiamare un nuovo endpoint [https://servizi.fatturapa.it/sm-scarico-file -> esitoRichiesta]

``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:typ="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
   <soapenv:Header/>
   <soapenv:Body>
      <typ:EsitoRichiestaRequest>
         <IdRichiesta>62609XXX</IdRichiesta>
      </typ:EsitoRichiestaRequest>
   </soapenv:Body>
</soapenv:Envelope>
```

La risposta sarà tipo questa
``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
   <soapenv:Body>
      <a:EsitoRichiestaResponse xmlns:a="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
         <Stato>ST01</Stato>
         <Tipo>SCARICO FATTURE</Tipo>
      </a:EsitoRichiestaResponse>
   </soapenv:Body>
</soapenv:Envelope>
```

Lo stato ST01 vuol dire in elaborazione, bisogna attendere 24/48 ore per una risposta completa
 Gli stati che assumerà saranno:
 - ST00 -> l’identificativo indicato nella chiamata non corrisponde a nessuna richiesta trasmessa dal provider che ne ha richiesto l’esito;
 - ST01 -> la richiesta è presente a sistema ed è in lavorazione, occorre ripetere più tardi l’operazione;
 - ST02 -> la richiesta è stata scartata per i motivi riportati nella lista di dettaglio degli errori e non verrà creato alcun file o archivio;
 - ST03 -> la richiesta è stata correttamente elaborata, sono stati prodotti i file richiesti e nella risposta viene restituito in allegato un file in formato xml contenente un elenco di identificativi e dettagli dei vari archivi scaricabili, nel payload è presente il numero di file generati ed il loro id, necessari per il punto successivo

> ### Hint per Soap-UI
> Per ottenere il file all'interno della risposta XML, al posto che nel payload, è necessario impostare nelle request properties (in basso sotto il navigator del progetto):
>| Param | Value |
>| - | - |
>| Enable MTOM | true |
>| Expand MTOM Attachments | true |

## Scarico file
Per scaricare i file bisognerà chiamare un nuovo endpoint [https://servizi.fatturapa.it/sm-scarico-file -> scaricoFile]
``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:typ="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
   <soapenv:Header/>
   <soapenv:Body>
      <typ:ScaricoFileRequest>
         <IdRichiesta>62609XXX</IdRichiesta>
         <IdFile>24582YYY</IdFile>
      </typ:ScaricoFileRequest>
   </soapenv:Body>
</soapenv:Envelope>
```

La risposta sarà tipo questa, per vedere il payload seguire [Hint per Soap-UI](hint-per-soap-ui) 
``` xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Body>
    <a:ScaricoFileResponse xmlns:a="http://ivaservizi.agenziaentrate.gov.it/docs/wsdl/ServiziMassivi/v1.0/types">
      <ArchivioFile>
        <NomeFile>612HHHHHH_FATT_0200000000.zip</NomeFile>
        <File>UEsDBBQACAgIAAYJbloAAAAAAAAAAAAAAAAXAAAASVQxM</File>
      </ArchivioFile>
    </a:ScaricoFileResponse>
  </soapenv:Body>
</soapenv:Envelope>
```

Prendendo il base64 del file e salvandolo in un file binario si può ottenere lo zip con:

`base64 -d archiviofile.base64 > 612HHHHHH_FATT_0200000000.zip` (linux)

`certutil -decode archiviofile.base64 612HHHHHH_FATT_0200000000.zip` (windows)
