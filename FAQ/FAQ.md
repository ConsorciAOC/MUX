# Preguntes freqüents

En aquesta plana pretenem fer un recull de dubtes i errors més freqüents en les operacions amb *MUX*, amb la descripció de la implementació necessaria per a la seva resolució.

# Errors en la descàrrega de la documentació associada a un assentament

En ocasions succeeix una casuística on un assentament amb numeració del vostre *Registre Principal*, no disposa de la documentació per a descarregar. 

En aquests casos, el problema moltes vegades és degut a que l'assentament s'ha realitzat al *Registre Auxiliar*. Per tant seria necessari revisar que veritablement aquest assentament no existeixi i que tingui relacionat un assentament al *Registre Auxiliar* d'EACAT (amb la corresponent numeració típica d'EACAT `[E/XXXXXX-2022 o S/XXXXXX-2022]`) si és així, la descàrrega de la documentació associada a aquest assentament, s'ha de fer a partir del número del *Registre Auxiliar*.

Us proposem doncs el següent flux com a bona pràctica, per tal d'evitar aquests errors:

1. Si després de `n` intents, no podeu fer la descàrrega de la documentació associada a un assentament del vostre *Registre Principal*, haureu de realitzar una petició `MUX_CONSULTA` d'aquell número d'assentament pel qual no trobeu la documentació. Les dades específiques d'aquesta petició haurien de tenir la següent forma:

```xml
<DatosEspecificos>
  <PeticioRegistre>
    <ConsultaAssentaments>
     <NumeroAssentament>ASSENTAMENT_REGISTRE_PRINCIPAL</NumeroAssentament>
    </ConsultaAssentaments>
  </PeticioRegistre>
</DatosEspecificos>
```

2. Si la consulta retorna dades per aquell assentament, seguir amb la política de reintents que tingueu desenvolupada per a la descàrrega d'aquell número d'assentament del vostre *Registre Principal*, i no caldria realitzar cap acció addicional.

3. Si per el contrari la consulta, no retorna dades, és a dir, teniu una resposta del registre amb la següent forma:

```xml
<DatosEspecificos>
  <RespostaRegistre>
    <Resultat codi="OK"/>
    <ConsultaAssentaments/>
  </RespostaRegistre>
</DatosEspecificos>
```

Vol dir que aquell assentament no és el correcte i s'ha de trobar l'assentament auxiliar corresponent, i consolidar-lo al vostre aplicatiu. Com ja sabeu, aquesta tasca la podeu fer amb una petició `MUX_CONSULTA` per franja horària i indicant que només mostri els assentaments que han anat a parar al *Registre Auxiliar*. Les dades específiques d'aquesta petició haurien de tenir la següent forma:

```xml
<DatosEspecificos>
  <PeticioRegistre>
    <ConsultaAssentaments>
      <DataInici>2022-XX-XXTXX:XX:XX</DataInici>
      <DataFi>2022-XX-XXTXX:XX:XX</DataFi>
      <Registre>Auxiliar</Registre>
    </ConsultaAssentaments>
  </PeticioRegistre>
</DatosEspecificos>
```
