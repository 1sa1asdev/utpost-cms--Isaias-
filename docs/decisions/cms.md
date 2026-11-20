# Beslutsdokument: CMS

*Ett av teamets beslutsdokument. Skrivs i M9. Högst en sida. Ni ska inte utvärdera marknaden, ni ska kunna försvara valet: varför det här räcker för Utpost, vad ni valde bort, och vad som hade fått er att byta.*

**Datum:**
**Beslut:** *(en eller två meningar: vilket innehåll bor i CMS:et, vilket CMS, och vad ligger kvar i databasen?)*

## Bakgrund
*Skuld 10: redaktionens innehåll låg som `body_html` i Postgres och sattes in med `v-html`. Redaktörerna mejlade ändringar till utvecklarna. Två meningar om vad det kostade, med ett exempel ur er egen kod.*

## Innehållsmodellen
*Era innehållstyper och fält. Fälten ska säga vad innehållet är, inte hur det ser ut.*

| Innehållstyp | Fält | Typ | Varför ett eget fält |
|---|---|---|---|
| | | | |

*Vad är innehåll och vad är data? Vilka fält ligger kvar i Postgres (till exempel längd och svårighetsgrad), och hur hittar API:et rätt post i CMS:et?*

## Alternativ
*Kursens val och ett alternativ ni valde bort. Det bortvalda ska vara ett riktigt alternativ: ett annat headless CMS, ett som ni driftar själva, eller Markdown i repot.*

| | Det vi valde | Det vi valde bort |
|---|---|---|
| Vad det är | | |
| Vem kan redigera, och hur | | |
| Utkast och förhandsgranskning | | |
| Vad det kostar oss (pengar, drift, inlåsning) | | |

## Motivering
*Varför räcker valet för Utpost just nu? Knyt till vem som redigerar och hur ofta.*

## Vad som hade fått oss att byta
*En eller två konkreta saker: en gräns i gratisnivån, ett krav från redaktionen, en lansering i ett land till.*

- **Taket på API-anrop.** Gratisnivån har ett hårt tak på 100 000 API-anrop i månaden, för alla API:er tillsammans (CDA, CPA och CMA). Inga extra anrop tillåts, och det går inte att betala för fler på gratisnivån. När taket är nått kan vi inte hämta något nytt från Contentful resten av månaden: API:et svarar med det senast hämtade, eller inga notiser alls efter en omstart. Med fler besökare, eller när guiderna flyttas in i Contentful och varje guidesida blir ett anrop till, kan taket bli för lågt. Gratisnivån är dessutom bara för test och lärande, inte kommersiell drift. Källa: https://www.contentful.com/help/admin/usage/usage-limit/ (se även https://www.contentful.com/pricing/)

## Konsekvenser
*Vad händer när CMS:et är nere? Hur länge dröjer en publicering innan den syns, och varför? Vad i koden måste skrivas om den dag ni byter CMS?*

**Tre nycklar, tre rättigheter.** Varje nyckel har bara den rättighet den behöver. Hade det funnits en enda nyckel med alla rättigheter hade en läcka gett någon rätt att både läsa utkast och skriva i vårt space.

| Nyckel | Vad den läser eller gör | Var den ligger |
|---|---|---|
| Delivery (`CONTENTFUL_DELIVERY_TOKEN`) | Läser bara publicerat innehåll (`cdn.contentful.com`) | `.env` på servern, i API:et. Aldrig i klienten |
| Preview (`CONTENTFUL_PREVIEW_TOKEN`) | Läser även utkast (`preview.contentful.com`) | `.env` på servern. Används bara när resolvern har kontrollerat att användaren är redaktör |
| Management (`CONTENTFUL_MANAGEMENT_TOKEN`) | Skriver i spacet (`api.contentful.com`) | Bara i vår lokala `.env`, för `npm run cms:setup`. Appen använder den aldrig och den ska inte till molnet |

`.env` är gitignorerad, så ingen nyckel committas. Space-id:t är ingen hemlighet.

**När Contentful inte svarar.** Med fel delivery-nyckel skriver API-loggen `cms: kunde inte hämta notiser (Contentful svarade 401) – svarar med en tom lista` (eller `det senast hämtade` om något redan låg i cachen). Startsidan fungerar ändå, men utan notiser. Det är ett medvetet val: notiserna är inte viktigare än guiderna, så ett CMS som är nere ska inte ta ner sidan.

**Från publicering till synlig notis, utan webhook.** Från några sekunder upp till ungefär en minut. API:et svarar på publicerade notiser ur minnet i `CMS_CACHE_SECONDS` (60 s) och frågar Contentful först när tiden gått ut. Vi mätte cirka 3 sekunder, men då hade cachen redan gått ut. Precis efter en sidvisning hade det tagit nära en minut. Delivery-API:et kan dessutom ge den gamla versionen i några sekunder efter en publicering. Det är skälet till webhooken.
